# DodoAppServiceMonitor

`DodoAppServiceMonitor` — это Custom Resource для управления мониторингом сервисов, развёрнутых через `DodoAppService`: автоматически создаёт дашборды в Grafana и настраивает алертинг.

## Основные возможности

- Автоматическое создание common dashboard'а для сервиса
- Настройка SLO-алертов с разделением на дневное и ночное время
- Включение стандартных алертов для MySQL, Kafka, Mongo, Redis, Pods и др.
- Создание кастомных алертов с юнит-тестами
- Маршрутизация info-алертов в отдельный канал Time

## Быстрый старт

Минимальный пример:

```yaml
apiVersion: k8s.paas.dodois.io/v1
kind: DodoAppServiceMonitor
metadata:
  name: my-service-monitor
  namespace: my-namespace
spec:
  serviceRef:
    name: my-service  # имя DodoAppService
  workspaceShortName: mysvc  # короткое имя из IaC, например: labelprinter → lprt
```

**Что произойдёт при применении этой конфигурации:**
- Создастся common dashboard для сервиса
- Никакие алерты по умолчанию **НЕ** включаются
- Для включения алертов нужно явно настроить секцию `alerting` (см. ниже)

## Common Dashboard

При создании `DodoAppServiceMonitor` автоматически создаётся дашборд в Grafana с базовыми метриками сервиса:

- **Ресурсы подов**: CPU и память
- **HTTP метрики**: количество запросов, статус-коды (2xx, 4xx, 5xx)
- **Производительность**: latency (p50, p95, p99)
- **Надёжность**: ошибки, рестарты подов

**Как найти дашборд:**

1. Откройте Grafana вашего кластера
2. Перейдите в папку с именем вашего namespace (например, `notifications`)
3. Найдите дашборд с именем: `<service-name> - common dashboard (v2)`

**Пример:** для сервиса `notifications` в namespace `notifications` дашборд будет называться **`notifications - common dashboard (v2)`**

## Спецификация (spec)

### Обязательные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `serviceRef.name` | string | Имя DodoAppService, для которого настраивается мониторинг |
| `workspaceShortName` | string | Короткое имя сервиса из IaC/Terraform. Найти можно в репозитории `infra-terraform-dubai` в файле `clouds/azure/services/<service_name>.libsonnet` в поле `shortname`. Пример: для сервиса `labelprinter` → `lprt` |

### Опциональные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `infoAlertsChannel` | string | Канал Time для алертов с severity=info. Используйте `name`, а не `display_name`. Если не указан, info-алерты игнорируются |
| `alerting` | object | Настройки алертинга |

## Секция alerting

### Стандартные алерты

Для включения стандартных алертов установите `enabled: true`.

> **Список алертов и их конфигурация**: [исходный код в репозитории оператора](https://github.com/dodopizza/infra.monitoring.observability-operator) (см. директорию `internal/alerts/`)

```yaml
spec:
  alerting:
    mysql:
      enabled: true
    kafka:
      enabled: true
      max_queue: 5000  # опционально, default: 5000, range: 1-5000
    mongo:
      enabled: true
      cpu_threshold: 70  # опционально, default: 70
    redis:
      enabled: true
    pods:
      enabled: true
    cronjob:
      enabled: true
    local_command_queue:
      enabled: true
```

| Тип | Описание | Дополнительные параметры |
|-----|----------|-------------------------|
| `mysql` | Алерты для MySQL | — |
| `kafka` | Алерты для Kafka | `max_queue` (1-5000, default: 5000) |
| `mongo` | Алерты для MongoDB | `cpu_threshold` (default: 70) |
| `redis` | Алерты для Redis | — |
| `pods` | Алерты для подов (restarts, OOM и т.д.) | — |
| `cronjob` | Алерты для CronJob | — |
| `local_command_queue` | Алерты для Local Command Queue | — |

#### Алерты для монолита

**⚠️ Внимание**: Включайте эти алерты только если понимаете что делаете. Неправильное использование может привести к дублированию алертов или флапам.

Монолит имеет специфическую логику алертинга. Перед включением ознакомьтесь с [исходным кодом алертов для монолита](https://github.com/dodopizza/infra.monitoring.observability-operator/blob/main/internal/alerts/monolith.go).

```yaml
spec:
  alerting:
    monolith:
      enabled: true
```

### SLO-алерты

> **Для большинства сервисов достаточно дефолтных настроек.**
> - **Хотите алерты на "девятки" (99.9% availability)?** → Просто укажите deployment и используйте дефолтные настройки
> - **Не хотите SLO-алерты?** → Установите `mode: Disabled` или не указывайте deployment в секции `slo`
> - **Хотите настроить кастомные burn rates?** → См. раздел "Параметры SLO для продвинутых пользователей" ниже

SLO настраиваются per-deployment. Ключ в map `slo` должен соответствовать имени deployment в DodoAppService.

```yaml
spec:
  alerting:
    slo:
      api:  # имя deployment
        day:
          mode: SLO
          settings:
            slo:
              ratio: 99.9
              burnRate5m: 10
              burnRate1h: 5
        night:
          mode: Max errors
          settings:
            maxErrors: 50
        runbookUrl: https://wiki.example.com/runbook/api
```

#### Временные окна

| Окно | Время (MSK) | Default mode | Default settings |
|------|-------------|--------------|------------------|
| `day` | 8:00 - 23:00 | SLO | ratio: 99.9, burnRate5m: 10, burnRate1h: 5 |
| `night` | 23:00 - 8:00 | Max errors | maxErrors: 50 |

#### Режимы (mode)

| Режим | Описание |
|-------|----------|
| `Disabled` | Алерты отключены |
| `SLO` | Алерт по SLO с burn rate |
| `Max errors` | Алерт по абсолютному числу ошибок |

#### Параметры SLO для продвинутых пользователей

> **Эти параметры нужны только тем, кто понимает концепцию burn rate. Остальным хватит дефолтов.**

| Параметр | Тип | Описание | Ограничения |
|----------|-----|----------|-------------|
| `ratio` | float | Целевой SLO (% успешных запросов) | 99 - 99.99 |
| `burnRate5m` | int | Во сколько раз ошибок больше чем при нормальной работе с заданным SLO (проверка за последние 5 минут). Чем выше значение, тем позже сработает алерт | >= 1 |
| `burnRate1h` | int | Во сколько раз ошибок больше чем при нормальной работе с заданным SLO (проверка за последний час). Чем выше значение, тем позже сработает алерт | >= 1 |

**Дополнительная информация**: [Multiwindow, Multi-Burn-Rate Alerts (Prometheus docs)](https://sre.google/workbook/alerting-on-slos/#6-multiwindow-multi-burn-rate-alerts)

#### Параметры Max errors

| Параметр | Тип | Описание |
|----------|-----|----------|
| `maxErrors` | int | Максимальное число ошибок за 5 минут |

### Кастомные алерты (extra)

Позволяют создавать собственные alert rules с возможностью добавления юнит-тестов.

```yaml
spec:
  alerting:
    extra:
      - alert: HighErrorRate
        expr: error_count > 100
        for: 1m
        severity: critical
        labels:
          team: backend
        annotations:
          summary: Too many errors detected
          description: Service has too many errors
```

#### Обязательные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `alert` | string | Имя алерта |
| `expr` | string | PromQL-выражение |
| `for` | string | Длительность до срабатывания (например: `1m`, `5m`, `1h`) |
| `severity` | enum | Уровень: `info`, `warning`, `critical` |

#### Опциональные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `labels` | map[string]string | Дополнительные labels для алерта. **Примечание**: оператор автоматически добавляет label `service` со значением из `spec.serviceRef.name` |
| `annotations` | map[string]string | Аннотации (summary, description, runbook_url и т.д.) |
| `tests` | array | Юнит-тесты для алерта |

### Юнит-тесты для кастомных алертов

Тесты позволяют проверить корректность alert rules до применения.

```yaml
spec:
  alerting:
    extra:
      - alert: HighErrorRate
        expr: error_count > 100
        for: 1m
        severity: critical
        labels:
          team: backend
        annotations:
          summary: Too many errors detected
          description: Service has too many errors
        tests:
          - name: When errors are low should not fire
            evalTime: 5m
            timeSeriesInterval: 15s
            timeSeries:
              - series: error_count
                values: 50x5
            expectedLabels: {}

          - name: When errors are high should fire alert
            evalTime: 5m
            timeSeriesInterval: 15s
            timeSeries:
              - series: error_count
                values: 200x5
            expectedLabels:
              service: my-service
              severity: critical
              team: backend
            expectedAnnotations:
              summary: Too many errors detected
              description: Service has too many errors
```

#### Поля теста

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | string | Название теста |
| `evalTime` | string | По прошествии этого времени с начала теста будет проверяться срабатывание алерта (например: `5m`, `10m`) |
| `timeSeriesInterval` | string | Интервал между точками данных (например: `15s`, `1m`) |
| `timeSeries` | array | Входные данные временных рядов |
| `expectedLabels` | map | Ожидаемые labels сработавшего алерта. Пустой `{}` означает, что алерт не должен сработать |
| `expectedAnnotations` | map | Ожидаемые annotations сработавшего алерта |

#### Формат values в timeSeries

| Формат | Описание | Пример |
|--------|----------|--------|
| Список значений | Пробелами разделённые значения | `50 50 50 50 50` |
| Повторение | `<value>x<count>` | `50x5` → `50 50 50 50 50` |
| Арифметическая прогрессия | `<start>+<step>x<count>` | `0+10x6` → `0 10 20 30 40 50 60` (7 значений: начальное + 6 шагов) |

## Полный пример

```yaml
apiVersion: k8s.paas.dodois.io/v1
kind: DodoAppServiceMonitor
metadata:
  name: notifications-monitor
  namespace: notifications
spec:
  serviceRef:
    name: notifications-service
  workspaceShortName: ntfs  # короткое имя из IaC (см. infra-terraform-dubai)
  infoAlertsChannel: notifications-info-alerts

  alerting:
    slo:
      api:
        day:
          mode: SLO
          settings:
            slo:
              ratio: 99.95
              burnRate5m: 10
              burnRate1h: 5
        night:
          mode: Max errors
          settings:
            maxErrors: 20
        runbookUrl: https://wiki.dodo.dev/notifications/runbook

      worker:
        day:
          mode: Max errors
          settings:
            maxErrors: 100
        night:
          mode: Disabled

    mysql:
      enabled: true
    kafka:
      enabled: true
      max_queue: 1000
    redis:
      enabled: true
    pods:
      enabled: true

    extra:
      - alert: NotificationDeliveryDelayed
        expr: histogram_quantile(0.99, rate(notification_delivery_duration_seconds_bucket[5m])) > 30
        for: 5m
        severity: warning
        annotations:
          summary: Notification delivery is slow
          description: 99th percentile of notification delivery time exceeds 30 seconds
        tests:
          - name: Normal latency should not fire
            evalTime: 10m
            timeSeriesInterval: 1m
            timeSeries:
              - series: 'notification_delivery_duration_seconds_bucket{le="10"}'
                values: 100x10
              - series: 'notification_delivery_duration_seconds_bucket{le="+Inf"}'
                values: 100x10
            expectedLabels: {}
```

## Status

После применения ресурса, контроллер обновляет поле `status`:

| Поле | Описание |
|------|----------|
| `phase` | Текущая фаза: `Deploying`, `Running`, `Failed` |
| `release.error` | Текст ошибки при фазе `Failed` |
| `release.started_at` | Время начала последнего reconcile |
| `release.finished_at` | Время завершения последнего reconcile |

### Диаграмма переходов фаз

```
     ┌─────────────┐
     │   Initial   │
     └──────┬──────┘
            │
            v
     ┌─────────────┐
  ┌──│  Deploying  │◄───────────┐
  │  └──────┬──────┘            │
  │         │                   │
  │         │ (успех)           │
  │         v                   │
  │  ┌─────────────┐            │
  │  │   Running   │────────────┤
  │  └─────────────┘            │
  │                             │
  │ (ошибка)    (изменение spec │
  │              или retry)     │
  │                             │
  v                             │
┌────────┐                      │
│ Failed │──────────────────────┘
└────────┘
```

**Пояснения:**
- При создании ресурса → `Deploying`
- При любом изменении spec → возврат в `Deploying`
- После ошибки при retry → возврат в `Deploying`
- Контроллер всегда проходит через `Deploying` при reconcile

### Проверка статуса

```bash
kubectl get dodoappservicemonitor -n my-namespace
# или короткое имя
kubectl get dasmon -n my-namespace
```

### Траблшутинг

**Если что-то пошло не так:**

1. **Проверьте фазу ресурса:**
   ```bash
   kubectl get dasmon my-service-monitor -n my-namespace
   ```

   Вывод покажет колонки: `NAME`, `SERVICE`, `PHASE`, `AGE`.

   - Если фаза `Failed` — переходите к шагу 2
   - Если фаза `Deploying` длительное время (>5 минут) — переходите к шагу 2
   - Если фаза `Running` — всё в порядке

2. **Посмотрите текст ошибки:**
   ```bash
   kubectl get dasmon my-service-monitor -n my-namespace -o yaml
   ```

   Ошибка будет в поле `status.release.error`. Или используйте:

   ```bash
   kubectl get dasmon my-service-monitor -n my-namespace -o jsonpath='{.status.release.error}'
   ```

3. **Посмотрите логи контроллера для детальной диагностики:**
   ```bash
   kubectl logs -n infra-monitoring deployment/observability-operator-controller-manager -f
   ```

   Найдите записи с вашим `serviceRef.name` и `namespace`.

4. **Если проблема непонятна:**

   Обратитесь к команде observability - Time канал: `#infra-platform`

**Типичные ошибки и решения:**

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `failed to get dodoappservice` | DodoAppService не найден | Проверьте что `spec.serviceRef.name` указывает на существующий DodoAppService в том же namespace |
| `failed to generate dashboard values` | Ошибка в конфигурации для генерации дашборда | Проверьте логи контроллера для деталей |
| `failed to generate alerts values` | Ошибка в настройках алертинга | Проверьте корректность `spec.alerting` (особенно SLO параметры: `ratio` должен быть 99-99.99, `burnRate*` >= 1) |
| `failed to generate alertmanager configs` | Ошибка в конфигурации alertmanager | Проверьте корректность `spec.infoAlertsChannel` |
| `failed to install/upgrade Helm release` | Ошибка деплоя через Helm | Проверьте логи контроллера, возможно проблема с доступом к кластеру или конфликт ресурсов |
| Зависание в `Deploying` | Контроллер не может завершить деплой | Проверьте логи контроллера и статус пода контроллера |

## Короткое имя

Для удобства можно использовать короткое имя `dasmon`:

```bash
kubectl get dasmon
kubectl describe dasmon my-service-monitor
```
