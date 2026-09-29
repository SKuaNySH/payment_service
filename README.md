# Обновление курсов валют и запуск в Docker

## Технологический стек

- Java 17 и Spring Boot — приложение и планировщик задач.
- Spring Cloud OpenFeign — HTTP-клиент для API курсов валют.
- Resilience4j Retry — повторные запросы при временных сбоях.
- `open.er-api.com` — источник курсов валют.
- Docker и Eclipse Temurin JDK 17 — упаковка и запуск приложения в контейнере.
- Gradle — сборка исполняемого JAR.

## Автоматическое обновление курсов валют

Добавлен ежедневный планировщик, который в 12:00 по времени `Asia/Almaty` запрашивает курсы валют у `open.er-api.com` с базовой валютой `EUR`.

Полученные курсы сохраняются в потокобезопасном хранилище как последний снимок с валютой-основой и временем получения. Если внешний API вернул пустой ответ, прежний снимок не перезаписывается. При сетевых ошибках и HTTP 5xx Resilience4j повторяет запрос до 5 раз с экспоненциальной задержкой, начиная с 2 секунд. Если повторы исчерпаны, ошибка записывается в лог.

Ключевые компоненты:

- `CurrencyRateFetcher` запускает обновление по расписанию.
- `ExternalCurrencyClient` запрашивает курсы через OpenFeign.
- `CurrencyServiceImpl` получает данные, проверяет ответ и обновляет снимок.
- `CurrencyRateStore` хранит последний снимок в памяти процесса.

Настройки находятся в `src/main/resources/application.yaml`:

| Параметр | Назначение |
|---|---|
| `currency.fetch.cron` | Расписание обновления (`0 0 12 * * ?`) |
| `currency.fetch.base-currency` | Базовая валюта (`EUR`) |
| `external.exchangeratesapi.base-url` | Адрес API курсов |
| `resilience4j.retry.instances.currencyRetry.maxAttempts` | Максимальное количество попыток |
| `resilience4j.retry.instances.currencyRetry.waitDuration` | Начальная задержка между попытками |

Снимок хранится только в памяти и сбрасывается при перезапуске приложения. Это обновление курсов не меняет существующий процесс конвертации платежей.

## Запуск приложения в Docker

Dockerfile использует образ Eclipse Temurin с JDK 17, копирует `build/libs/service.jar` в `/app/app.jar` и запускает его командой `java -jar`.

Из корня репозитория соберите JAR и Docker-образ:

```powershell
.\gradlew.bat bootJar
docker build -t payment-service .
```

Запустите контейнер, пробросив порт приложения:

```powershell
docker run --rm -p 9080:9080 payment-service
```
