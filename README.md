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








