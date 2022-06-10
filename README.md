# Мониторинг и логирование K8S
Цель — настроить мониторинг и логирования для кластера Kubernetes. Графический интерфейс — Grafana и Kibana. Для оповещения задействуем email и Telegram.

## 1. Мониторинг K8S 

Мониторинг приложений и серверов приложений — важная часть DevOps-культуры. Существует потребность постоянно мониторить состояние приложения и серверов, загрузку центрального процессора, потребление памяти, дисковую утилизацию и т.д. Также необходимо получать уведомления, если у сервера заканчивается доступная память или приложение перестает отвечать на запросы, что позволит предотвратить проблемы.

В этом проекте для монитринга кластер K8S реализовано решение с помощью Prometheus — инструмент для одновременного мониторинга десятков тысяч служб.

Prometheus — популярный проект с открытым исходных кодом, большая часть компонентов которого написана на Golang, а часть — на Ruby. 

### Установка сбора и отображение метрик

#### 1.1. Prometheus

Создать пространство имён, склонировать Git-репозиторий и развернуть Prometheus:

```
$ kubectl create ns monitoring
$ git clone git@github.com:GaripovAynur/Observability_OTUS.git
$ cd Chart/prometheus
$ helm show values prometheus-community/prometheus
$ helm install prometheus prometheus-community/prometheus
```
Установленные компоненты

**Kube-state-metrics** — собирает метрики со всех узлов Kubernetes, обслуживаемых Kubelet через Summary API

**Node-exporter** — собирает метрики о вычислительных ресурсах Node в Kubernetes.

**Alertmanager** — инструмент для обработки оповещений. Он устраняет дубликаты, группирует и отправляет нотификации

*****Архитиктура сбора метрик*****:
#
![Prometheus](https://phoenixnap.com/kb/wp-content/uploads/2021/04/example-prometheus-server-elements-inner-workings.png)

**Извлечение метрик пода с помощью аннотаций**

На этом примере используется конфигурация по умолчанию, которая заставляет prometheus фильтровать различные типы ресурсов kubernetes при условии, что они имеют правильные аннотации. Опишем, как настроить парсинг endpoints и node; для получения информации о том, как можно очистить другие типы ресурсов.

Чтобы Prometheus выбираль только нужные endpoints, необходимо добавить аннотации к endpoints и node, как пуказано ниже:

Пример 1.

```yaml
   scrape_configs:
     job_name: kubernetes-apiservers # Будет собирать метрики с API серверов
     kubernetes_sd_configs:
     - role: endpoints # Запрашивает у API endpoints (node, ingress и т.д.)
     relabel_configs: # Фильтрация, оставить только которые соответствуют API серверам
     - action: keep
       regex: default;kubernetes;https # Значении для labels
       source_labels:
       - __meta_kubernetes_namespace
       - __meta_kubernetes_service_name
       - __meta_kubernetes_endpoint_port_name
```
Пример 2.
```yaml
job_name: kubernetes-nodes
     kubernetes_sd_configs:
     - role: node
     relabel_configs:
     - action: labelmap # Позволяет замапить один label на другой (подставить какие то значении как пример). Мы собираем все node
       regex: __meta_kubernetes_node_label_(.+)
     - replacement: kubernetes.default.svc:443
       target_label: __address__ # Заменяем адрес на kubernetes.default.svc:443 (Адрес API сервера), чтобы за метриками нодов ходил на API сервер
     - regex: (.+)
       replacement: /api/v1/nodes/$1/proxy/metrics # Дальше подставляем kubernetes.default.svc:443/api/v1/nodes/$1/proxy/metrics в конечном итоге получаем метрики для __address__=”172.16.16.7:10250” - просто так пойти не можем, по этому нужна авторизация
       source_labels:
       - __meta_kubernetes_node_name
       target_label: __metrics_path__
     scheme: https
     tls_config:
       ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
       insecure_skip_verify: true

```

### 1.2. Grafana
Grafana — это платформа с открытым исходным кодом для визуализации, мониторинга и анализа данных.

***Дашборд Grafana***

+ Дашборд — набор отдельных панелей, размещенных в сетке с набором переменных (например, имя сервера, приложения и датчика). Изменяя переменные, можно переключать данные, отображаемые на дашборде (например, данные с двух отдельных серверов). Все дашборды можно настраивать, а также секционировать и фрагментировать представленные в них данные в соответствии с потребностями. В проекте Grafana участвует большое сообщество разработчиков кода и пользователей, поэтому существует большой выбор готовых дашбордов для разных типов данных и источников.

![Grafana](https://raw.githubusercontent.com/srinisbook/images/master/Prometheus-grafana.png)


Перейти в папку grafana и развернуть Grafana:

```
$ git clone git@github.com:GaripovAynur/Observability_OTUS.git
$ cd Chart/gafana
$ helm show values gafanay/gafana
$ helm install gafana gafana/gafana
```
После установки подключаем Dashboard 
``` cd Chart/gafana ```
***K8sDashboard.json***


## 2. Логирование K8S Elastic + Fluent Bit + Kibana (EFK Stack)

Fluent Bit —  легковесный и производительный агент-сборщик

Если верить разработчикам, то Fluent Bit более, чем в 100 раз лучше по производительности, чем Fluentd: «там, где Fluentd потребляет 20 Мб ОЗУ, Fluent Bit будет потреблять 150 Кб» — прямая цитата из документации. Глядя на это, Fluent Bit стали использовать чаще.


У Fluent Bit меньше возможностей, чем у Fluentd, но основные потребности он закрывает.

Схема работы стека EFK: агент собирает логи со всех подов (как правило, это DaemonSet, запущенный на всех серверах кластера) и отправляет в хранилище (Elasticsearch, PostgreSQL или Kafka). Kibana подключается к хранилищу и достаёт оттуда всю необходимую информацию.

![EFK](https://miro.medium.com/max/1400/1*laK4tNtVOTntZKcSK5ITXg.png)


### Возможности Fluent Bit

Fluent Bit логически можно поделить на 6 модулей, на часть модулей можно навесить плагины, которые расширяют возможности Fluent Bit.

Модуль **Input** собирает логи из файлов, служб systemd и даже из tcp-socket (надо только указать endpoint, и Fluent Bit начнёт туда ходить). Этих возможностей достаточно, чтобы собирать логи и с системы, и с контейнеров.

 Использую плагины tail (его можно натравить на папку с логами).


Модуль **Parser** приводит логи к общему виду. По умолчанию логи Nginx представляют собой строку. С помощью плагина эту строку можно преобразовать в JSON: задать поля и их значения. С JSON намного проще работать, чем со строковым логом, потому что есть более гибкие возможности сортировки.


Модуль **Filter** На этом уровне отсеиваются ненужные логи. Например, на хранение отправляются логи только со значением “warning” или с определёнными лейблами. Отобранные логи попадают в буфер.


Модуль **Buffer** У Fluent Bit есть два вида буфера: буфер памяти и буфер на диске. Буфер — это временное хранилище логов, нужное на случай ошибок или сбоев. Всем хочется сэкономить на ОЗУ, поэтому обычно выбирают дисковый буфер. Но нужно учитывать, что перед уходом на диск логи всё равно выгружаются в память.


Модуль **Routing/Output** содержит правила и адреса отправки логов. Как уже было сказано, логи можно отправлять в Elasticsearch, PostgreSQL или, например, Kafka.

Файлы деплоя для fluentbit в kubernetes

~~~~bash
$ kubectl create ns monitoring
$ git clone git@github.com:GaripovAynur/Observability_OTUS.git
$ cd Chart/fluentbit
$ kubectl creat -f ./
~~~~


* 01-flb-acc.yaml - ServiceAccount, ClusterRole, ClusterRoleBinding, Secret (access to elasticsearch)
* 02-flb-cm.yaml - ConfigMap. Конфигурационные файлы fluentbit.
* 03-flb-services.yaml - Service и Endpoints для доступа ко внешнему (за пределами кластера k8s) elasticserach.
* 04-flb-ds.yaml - DaemonSet

```yaml
  fluent-bit.conf: |

    [SERVICE] # Сексия
        Flush         1
        Log_Level     info 
        Daemon        off           # Режим демона, когда контейнер умерает, fluent-bit тоже.
        Parsers_File  parsers.conf  # Имя парсер файла
        HTTP_Server   On            # Включаем HTTP Server, может получить метрики двух видов Prometheus и JSON
        HTTP_Listen   0.0.0.0 
        HTTP_Port     2020
    @INCLUDE input-kubernetes.conf   # Включаем 3 доп.файла см. ниже (файлы могут иметь любое название)
    @INCLUDE filter-kubernetes.conf
    @INCLUDE output-elasticsearch.conf
 
  input-kubernetes.conf: | #

    [INPUT]
        Name              tail  # Режим просмотр с конца файла
        Tag               app.* # Метим тегом который будеть дальше отсылаться для обработки фильтра
        Path              /var/log/containers/app-*.log  # Указываю где находятся логи
        Parser            docker
        DB                /var/log/flb-example-app.db # Сохраняет какие логи были прочитаны, в случае сбоя продолжит с того места где остановился.
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10
    [INPUT]
        Name tail
        Tag sysapp.gen.log.messages
        Parser sys_log_file
        Path /var/log/messages
        db /var/log/messages.db
  
  filter-kubernetes.conf: | #

    [FILTER]
        Name                kubernetes
        Match               app.* # Говорим какой поток мы отбираем (INPUT Tag) 
        Kube_URL            https://kubernetes.default.svc.cluster.local:443 # Чтобы эту ифнормацию получить должен быть доступ к API
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     app.var.log.containers. # Должны писать "Match" и дальше var.log.containers.
        Merge_Log           On  # Это нужно когда JSON логи, получаем красивый лог
        Merge_Log_Key       log_processed # Здесь описывается с какой строкой будет начинаться каждое поля внутри одного JSON файла 
        K8S-Logging.Parser  Off # 
        K8S-Logging.Exclude Off
    [FILTER]
        Name modify
        Match sysapp.gen.log.messages
        Set app syslog # Тоже самое Merge_Log_Key, см.выше 
        Set file /var/log/messages

  output-elasticsearch.conf: | #

    [OUTPUT]
        Name            es
        Match           app.*
        Host            ${FLUENT_ELASTICSEARCH_HOST}
        Port            ${FLUENT_ELASTICSEARCH_PORT}
        HTTP_User       ${FLUENT_ELASTICSEARCH_USER}
        HTTP_Passwd     ${FLUENT_ELASTICSEARCH_PASSWORD}
        Logstash_Format On # ELASTICSEARCH говорим что отправлеяем в формате Logstash
        Logstash_Prefix app # Индекс будет называться app
        Replace_Dots    On
        Retry_Limit     False # Отправляем ELASTICSEARCH пока не получит данные
    [OUTPUT]
        Name            es
        Match           sysapp.gen.log.*
        Host            ${FLUENT_ELASTICSEARCH_HOST}
        Port            ${FLUENT_ELASTICSEARCH_PORT}
        HTTP_User       ${FLUENT_ELASTICSEARCH_USER}
        HTTP_Passwd     ${FLUENT_ELASTICSEARCH_PASSWORD}
        Logstash_Format On
        Logstash_Prefix sysapp-gen-log
        Replace_Dots    On
        Retry_Limit     False
  
  parsers.conf: | #

    [PARSER]
        Name        docker
        Format      json # Т.к. json особо парсит не нужно
        Time_Key    time # Указываем формат времени см. Time_Format 
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On
    [PARSER]
        Format regex
        Name sys_log_file # Что парсим, берем INPUT Parser
        Regex (?<message>(?<time>[^ ]*\s{1,2}[^ ]*\s[^ ]*)\s(?<host>[a-zA-Z0-9_\/\.\-]*)\s.*)$ # Определяем регулярное выражение.
        Time_Format %b %d %H:%M:%S
        Time_Keep Off # Поле Time дальше не пропускаю
        Time_Key time # Указываю в какому поле время 
        Time_Offset +0300 # Время МСК

```

За основу были использованы файлы из проекта https://github.com/fluent/fluent-bit-kubernetes-logging



