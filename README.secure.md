# Сборка с проверкой Critical / High

Выполняйте команды из D:\images\kafka_exporter.

```powershell
docker build --pull -f Dockerfile.secure -t kafka_exporter:secure .
docker run --rm kafka_exporter:secure --version
```

Используйте именно Dockerfile.secure. Исходный Dockerfile и make docker сохраняют прежнюю схему с готовым бинарником.

Dockerfile.secure собирает статический Linux-бинарник из текущих исходников с Go 1.27.2, проверяет контрольные суммы модулей и запускает существующие тесты. go.mod/go.sum фиксируют golang.org/x/net v0.60.0. Образы сборки и исполнения закреплены digest. Итоговый образ основан на Distroless static Debian 13 и запускается с UID/GID 65534, как прежний nobody. Он содержит системные CA-сертификаты и не содержит shell. Собственные CA и ключи по-прежнему нужно монтировать и передавать флагами exporter.

Тест TestSmoke требует Kafka на localhost:9092 и пропускается, если Kafka недоступна. Модульные тесты используют mock-брокеры; успешная сборка не заменяет проверку подключения к вашей Kafka, TLS/SASL и прав на смонтированные секреты.

## Docker Scout

Кэш, временные файлы и отчёты хранятся внутри проекта и исключены из контекста сборки.

```powershell
$env:DOCKER_SCOUT_CACHE_DIR = Join-Path $PWD '.security-cache\scout'
$env:TEMP = Join-Path $PWD '.security-cache\tmp'
$env:TMP = $env:TEMP
New-Item -ItemType Directory -Force -Path $env:TEMP,'security-reports' | Out-Null

docker scout cves local://kafka_exporter:secure --details --output security-reports/secure.txt

# Код 0: нет Critical/High; код 2: найдены Critical/High.
# Другие ошибки также требуют проверки, а не считаются успешным сканированием.
docker scout cves local://kafka_exporter:secure --only-severity critical,high --exit-code
if ($LASTEXITCODE -ne 0) { throw 'Проверка Docker Scout не пройдена' }
```

Проверка не исключает базовый образ и уязвимости без исправлений. Результат относится к конкретному образу и актуальной на момент проверки базе Scout. Перед публикацией повторяйте сканирование; при новых исправлениях обновляйте версии Go, модулей и закреплённые digest базовых образов.

Исходный отчёт: security-reports/baseline.txt. Итоговый отчёт: security-reports/secure.txt.

Источники: [исправления Go 1.27.2](https://go.dev/doc/devel/release#go1.27.2), [Distroless](https://github.com/GoogleContainerTools/distroless), [docker scout cves](https://docs.docker.com/reference/cli/docker/scout/cves/).

## Проверка запуска, tar и Kubernetes

Образ проверен с настоящей Kafka 4.1.0: HTTP `/`, `/healthz`, `/metrics` вернули 200, Kafka-метрики показали один брокер, две партиции тестового топика и нулевой lag после отправки и чтения трёх сообщений. Использовалось PLAINTEXT-соединение; TLS/SASL будущего внешнего кластера ещё нужно проверить отдельно. Подробности — в `security-reports/runtime.json` и `runtime-metrics.prom`.

Готовый архив: `kafka_exporter-secure-linux-amd64.tar` (рядом SHA256). Импорт: `docker load --input kafka_exporter-secure-linux-amd64.tar`.

[Настройка Kubernetes для внешней Kafka: ConfigMap, Deployment, Service, TLS/SASL](deploy/external-kafka/README.md).
