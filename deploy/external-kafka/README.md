# kafka_exporter в Kubernetes для внешней Kafka

Образ `kafka_exporter:secure` собран для **linux/amd64**. Примеры используют namespace `monitoring`. Здесь разворачивается только exporter; брокеры Kafka находятся вне Kubernetes.

## 1. Доставить образ

Архив в корне проекта: `kafka_exporter-secure-linux-amd64.tar`. Рядом находится SHA256.

Для Docker:

```powershell
Get-FileHash .\kafka_exporter-secure-linux-amd64.tar -Algorithm SHA256
Get-Content .\kafka_exporter-secure-linux-amd64.tar.sha256
docker load --input .\kafka_exporter-secure-linux-amd64.tar
```

Для Kubernetes проще загрузить образ в ваш registry:

```sh
docker tag kafka_exporter:secure registry.example.net/monitoring/kafka-exporter:1.10.0-secure
docker push registry.example.net/monitoring/kafka-exporter:1.10.0-secure
```

Затем заменить `image` в `deployment.yaml` на этот адрес. Для закрытого registry добавить `imagePullSecrets` вашей инфраструктуры.

Без registry импортируйте tar **на каждый узел**, на который может попасть Pod. Например, для обычного containerd на Linux:

```sh
sudo ctr --namespace k8s.io images import kafka_exporter-secure-linux-amd64.tar
```

Для kind используйте `kind load image-archive`, для других дистрибутивов — механизм импорта их container runtime. `docker load` на рабочем компьютере сам по себе не загружает образ на Kubernetes-узлы. В Deployment оставлен `imagePullPolicy: IfNotPresent`; в полностью изолированном окружении после импорта можно установить `Never`. Имя загруженного образа должно совпадать с `image`.

## 2. Настроить ConfigMap

В `configmap.yaml` заменить `KAFKA_BOOTSTRAP_1/2/3` на реальные `host:port` без `http://`. Каждый адрес передаётся отдельным `--kafka.server`. Если брокер один, удалить лишние аргументы из Deployment и соответствующие ключи ConfigMap. `KAFKA_VERSION` — версия протокола Kafka; `3.9.0` приведена как пример и использовалась при локальной проверке с брокером Kafka 4.1.0. Для вашего кластера укажите поддерживаемую им версию.

`KAFKA_LABELS` добавляет к Kafka-метрикам метку кластера. `TOPIC_FILTER` и `GROUP_FILTER` — регулярные выражения: можно ограничить сбор нужными топиками и группами. Количество workers, ресурсы и таймауты нужно подобрать под размер Kafka.

ConfigMap передаёт значения через environment; Kubernetes подставляет их в `args` с синтаксисом `$(NAME)`. Exporter не читает произвольный YAML-конфиг. В образе нет shell, поэтому `sh -c` не используется. Булевы флаги передаются отдельно: `--tls.enabled` или `--no-tls.enabled`; форма `--tls.enabled=false` этим бинарником не поддерживается.

### Выбрать режим соединения

| Режим Kafka | TLS_FLAG | SASL_FLAG | SASL_MECHANISM |
| --- | --- | --- | --- |
| PLAINTEXT | `--no-tls.enabled` | `--no-sasl.enabled` | `plain` (не используется) |
| SSL / TLS | `--tls.enabled` | `--no-sasl.enabled` | `plain` (не используется) |
| SASL_SSL | `--tls.enabled` | `--sasl.enabled` | `plain`, `scram-sha256` или `scram-sha512` |
| SASL_PLAINTEXT | `--no-tls.enabled` | `--sasl.enabled` | механизм вашего listener |

Порт определяется конфигурацией Kafka, а не этой таблицей. В базовом примере выбран PLAINTEXT. Для SASL_SSL обычно нужно поменять bootstrap-порты, включить TLS и SASL и создать Secret с учётными данными.

### Учётные данные SASL

Пароли хранить в Secret, не в ConfigMap. Создать локальный файл `.local/kafka-exporter-auth.env`:

```dotenv
username=exporter
password=REPLACE_WITH_REAL_PASSWORD
```

Создать Secret в том же namespace:

```sh
kubectl -n monitoring create secret generic kafka-exporter-auth --from-env-file=.local/kafka-exporter-auth.env
```

Deployment передаёт эти значения через `SASL_USER_NAME` и `SASL_USER_PASSWORD`; exporter поддерживает эти имена. Пароль не передаётся в аргументах процесса. Включение SASL требует корректного Secret, хотя ссылки помечены optional для работы базового PLAINTEXT-примера.

### TLS и собственный CA

Если серверные сертификаты подписаны вашим CA, создать Secret:

```sh
kubectl -n monitoring create secret generic kafka-exporter-tls --from-file=ca.crt=.local/ca.crt
```

В ConfigMap указать `TLS_FLAG: --tls.enabled` и `TLS_CA_FILE: /etc/kafka/tls/ca.crt`. Для CA из системного набора образа `TLS_CA_FILE` можно оставить пустым. Имена брокеров должны соответствовать SAN сертификатов. Проверка сертификатов остаётся включённой.

Для mutual TLS создать Secret сразу с сертификатом и ключом клиента:

```sh
kubectl -n monitoring create secret generic kafka-exporter-tls --from-file=ca.crt=.local/ca.crt --from-file=tls.crt=.local/client.crt --from-file=tls.key=.local/client.key
```

И дополнительно задать в ConfigMap:

```yaml
TLS_CERT_FILE: /etc/kafka/tls/tls.crt
TLS_KEY_FILE: /etc/kafka/tls/tls.key
```

Secret монтируется с правами 0440, Pod использует UID/GID/fsGroup 65534. При включении TLS с файловыми путями соответствующие файлы должны существовать в Secret.

## 3. Проверить сеть и права Kafka

Из Pod должны разрешаться DNS-имена и быть доступны порты **всех брокеров из `advertised.listeners`**, а не только bootstrap. Bootstrap возвращает метаданные, затем exporter обращается к адресам брокеров из них. Если Kafka рекламирует внутренние имена или localhost, внешний listener Kafka нужно настроить на адреса, достижимые из Kubernetes.

Учитывайте маршрутизацию, firewall, egress NetworkPolicy и DNS. Kafka-пользователю нужны права на просмотр метаданных топиков, consumer groups и их offsets; при использовании Kafka ACL обычно требуются Describe на нужные topics/groups и соответствующий доступ на уровне cluster. Точный набор зависит от версии Kafka и authorizer. Exporter не должен получать права записи сообщений ради сбора метрик.

## 4. Применить манифесты

Сначала создать namespace:

```sh
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
```

Затем при необходимости создать Secrets из предыдущего раздела, заменить адреса и образ, применить:

```sh
kubectl -n monitoring apply -f deploy/external-kafka/configmap.yaml
kubectl -n monitoring apply -f deploy/external-kafka/deployment.yaml
kubectl -n monitoring apply -f deploy/external-kafka/service.yaml
kubectl -n monitoring rollout status deployment/kafka-exporter --timeout=300s
kubectl -n monitoring logs deployment/kafka-exporter
```

После изменения ConfigMap или Secret перезапустить Pods: environment не обновляется в уже запущенном контейнере.

```sh
kubectl -n monitoring rollout restart deployment/kafka-exporter
kubectl -n monitoring rollout status deployment/kafka-exporter --timeout=300s
```

Deployment использует `/healthz` для startup/readiness/liveness. Этот endpoint проверяет HTTP-процесс и всегда отвечает `ok` после запуска; он **не проверяет непрерывную доступность Kafka**. При недоступной Kafka на старте exporter завершится до открытия HTTP. Для контроля фактического сбора проверяйте `/metrics`, Kafka-метрики и логи, а не только readiness.

## 5. Проверить метрики и подключить Prometheus

```sh
kubectl -n monitoring port-forward service/kafka-exporter 9308:9308
```

В другом терминале:

```sh
curl http://127.0.0.1:9308/healthz
curl http://127.0.0.1:9308/metrics
```

На Windows можно использовать `Invoke-WebRequest -UseBasicParsing`. Ожидаются `kafka_brokers`, метрики топиков и, при наличии committed offsets и нужных прав, `kafka_consumergroup_*`. Наличие только Go/process-метрик не подтверждает успешный сбор Kafka.

Пример задания Prometheus, запущенного в том же Kubernetes-кластере:

```yaml
scrape_configs:
  - job_name: external-kafka
    scrape_interval: 60s
    scrape_timeout: 55s
    static_configs:
      - targets:
          - kafka-exporter.monitoring.svc.cluster.local:9308
```

Если cluster DNS domain отличается, заменить `cluster.local`. Аннотации Service работают только при настроенном discovery/relabeling Prometheus; сами по себе они не включают сбор. При использовании Prometheus Operator создайте ServiceMonitor с selector `app.kubernetes.io/name: kafka-exporter` и endpoint `port: metrics`, учитывая selectors вашего Prometheus.

## Подтверждённая локальная проверка

Образ `sha256:27f2619e96bb8ec20b21ccba22daa14f4df4d01738baec79e353c2e565cd0380` проверен с настоящей временной Kafka `apache/kafka:4.1.0` по PLAINTEXT. HTTP `/`, `/healthz`, `/metrics`: 200. Создан топик `exporter-smoke` с двумя партициями, отправлены и прочитаны три сообщения; `kafka_brokers = 1`, `kafka_topic_partitions = 2`, lag тестовой группы = 0. Контейнер работал без root, с read-only root filesystem и без capabilities. Тестовые контейнеры и сеть после проверки удалены.

Результаты: `security-reports/runtime.json`, `runtime-metrics.prom`, `runtime-exporter.log`. Воспроизводимый стенд: `dev/docker-compose.secure-smoke.yml`. Проверка не подтверждает TLS/SASL и сеть вашей будущей Kafka. Манифесты подготовлены как примеры и в реальный Kubernetes не применялись.

Источники: [ConfigMap и подстановка в args](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/), [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/), [HTTP probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/), [Docker image save](https://docs.docker.com/reference/cli/docker/image/save/), [официальный Kafka Docker image](https://hub.docker.com/r/apache/kafka).
