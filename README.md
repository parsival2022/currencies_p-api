# Acme Sports EUR Sales Digest

A single Mule 4 application that reads the latest sales CSV on a schedule, enriches each sale with country data, converts local amounts to EUR, aggregates the batch, and posts the resulting digest to a Treasury HTTP endpoint.

The application does not expose an inbound API. Its main flow is scheduler-driven:

```text
schedule -> read CSV -> enrich countries -> convert to EUR -> aggregate -> POST digest
```

## Requirements

- Java 17
- Maven 3.8.1 or later (3.9.x recommended)
- Mule Runtime 4.9.11
- Anypoint Studio with Mule 4 support, or Maven with access to the MuleSoft repositories
- A REST Countries v5 API token
- Network access to REST Countries, Frankfurter, and the configured Treasury endpoint

## Input CSV

Set `salesFilepath` in `src/main/resources/properties/properties-local.yaml` to the absolute path of the sales file. The application reads that file on every scheduled run.

The CSV must have this header:

```csv
sale_id,sale_date,country_code,currency,amount_local,customer_email
```

Field expectations:

| Field | Format |
| --- | --- |
| `sale_id` | Unique sale identifier |
| `sale_date` | ISO date, `YYYY-MM-DD` |
| `country_code` | ISO 3166-1 alpha-2 code, for example `GB` |
| `currency` | ISO 4217 code, for example `GBP` |
| `amount_local` | Numeric local-currency amount |
| `customer_email` | Customer email address |

## Local configuration

Shared settings are in `src/main/resources/properties/common.yaml`. Environment-specific settings are loaded from `src/main/resources/properties/properties-${mule.env}.yaml`; the project defaults `mule.env` to `local`.

Before running locally:

1. Edit `src/main/resources/properties/properties-local.yaml`.
2. Set `salesFilepath` to the input CSV's absolute path.
3. Set `restCountries.apiToken` to a Mule Secure Properties encrypted value.
4. Supply the matching `encryption.key` as a runtime property. The repository contains a development default in `global.xml` so the provided local encrypted value can be opened; replace both values before using the project outside this exercise.

Do not commit a plaintext API token or a production encryption key.

### Important properties

| Property | Default | Purpose |
| --- | --- | --- |
| `referenceCurrency` | `EUR` | Digest currency and FX quote currency |
| `salesFilepath` | Environment-specific | Absolute path of the source CSV |
| `restCountries.*` | REST Countries v5 | Country API connection |
| `frankfurter.*` | Frankfurter v2 | FX API connection |
| `treasury.*` | `https://httpbin.org` | Digest destination |
| `*.responseTimeout` | `50000` ms | HTTP response timeout |
| `*.retry.maxRetries` | `5` | Maximum retry count |
| `*.retry.msBetweenRetries` | `30000` ms | Delay between retry attempts |
| `cache.countries.entryTtlMinutes` | `60` | Country response TTL |
| `cache.countries.maxEntries` | `300` | Maximum cached country responses |
| `cache.currencies.entryTtlMinutes` | `15` | FX response TTL |
| `cache.currencies.maxEntries` | `100` | Maximum cached FX responses |
| `cache.expirationIntervalMinutes` | `5` | Object Store expiry scan interval |

### Scheduler frequency

The default schedule is once per hour. It is defined in `src/main/mule/implementation.xml`:

```xml
<fixed-frequency frequency="1" timeUnit="HOURS"/>
```

For quicker local testing, temporarily change `timeUnit` to `MINUTES` or `SECONDS`. Restore the hourly value before packaging the submission.

## Running locally

### Anypoint Studio

1. Import the repository as an existing Mule project.
2. Confirm that the project uses Java 17 and Mule Runtime 4.9.11.
3. Complete the local configuration above.
4. Run the project as a Mule application.
5. Watch the console for the correlation ID, rows read, row processing, publishing, and completion messages.

The scheduler starts the flow automatically. There is no HTTP endpoint to invoke.

### Maven

Run the application from the repository root:

```shell
mvn mule:run -Dmule.env=local -Dencryption.key=<your-key>
```

Build the deployable Mule application:

```shell
mvn clean package
```

The package is written to:

```text
target/currencies_p-api-1.0.0-SNAPSHOT-mule-application.jar
```

## External integrations

| System | Request | Authentication |
| --- | --- | --- |
| REST Countries | `GET /countries/v5/code?q={country_code}` | API token in the `Authorization` header |
| Frankfurter | `GET /v2/rates?base={currency}&quotes=EUR` | None |
| Treasury test sink | `POST /post` | None |

Sales already denominated in EUR bypass Frankfurter, retain their local amount, and use an FX rate of `1.0`. Other amounts are calculated as `amount_local * rate`, where the API rate is requested from the sale currency to EUR.

Country responses are cached by uppercase country code. FX responses are cached by source and reference currency. Both caches are synchronized, in-memory Object Stores and therefore reset when the application restarts.

## Digest payload

The Treasury request contains the generated timestamp, batch date range, totals, country and currency summaries, and the enriched records. A representative payload is:

```json
{
  "generated_at": "2026-08-27T12:00:00Z",
  "period_start": "2026-01-01",
  "period_end": "2026-01-03",
  "total_sales_count": 3,
  "total_amount_eur": 190.0,
  "by_country": [
    {
      "country_code": "GB",
      "country_name": "United Kingdom",
      "region": "Europe",
      "sales_count": 2,
      "amount_eur": 90.0
    }
  ],
  "by_currency": [
    {
      "currency": "GBP",
      "sales_count": 2,
      "amount_eur": 90.0,
      "fx_rate": 1.2
    }
  ],
  "data": []
}
```

`period_start` and `period_end` are the earliest and latest non-empty `sale_date` values in the batch. The `data` array contains the original rows plus `country_name`, `region`, `fx_rate`, and `amount_eur`.

The current Treasury endpoint is HTTPBin. A successful run is complete only after HTTPBin acknowledges the POST.

## Resilience, errors, and logging

- Each external request has a configurable response timeout and an `until-successful` retry policy.
- Country and FX lookups are cached to reduce repeated calls within and across scheduled runs.
- Known HTTP client errors are mapped to integration-specific error types.
- Server, connectivity, retry-exhaustion, transformation, and publish errors propagate to the main scheduler error handler, so a failed run is not reported as successful.
- If enrichment or FX conversion fails, the Treasury POST is not executed.
- INFO logs include the correlation ID and record-level progress without printing the full digest.
- The global error log includes the correlation ID, error type, current sale ID, upstream path, and description when available.

## Tests

Run the MUnit suite with:

```shell
mvn clean test
```

The suite is in `src/test/munit/sales-digest-test-suite.xml` and uses mocked file and HTTP connectors, so it does not call the live services.

| Test | Coverage |
| --- | --- |
| `implementationFlow-happy-path-test` | EUR passthrough, GBP conversion, period and total aggregation, country/currency summaries, generated timestamp, caching call counts, and one Treasury delivery |
| `implementationFlow-fx-error-prevents-publish-test` | Propagation of a Frankfurter server error and verification that Treasury is never called |

Surefire-compatible results are generated under `target/surefire-reports`.

## Project structure

```text
src/main/mule/
  implementation.xml             Main scheduled flow and enrichment logic
  global.xml                     HTTP clients, properties, and cache strategies
  error-handler.xml              Global scheduled-run error handler
  utils/                         External HTTP request and retry subflows
src/main/resources/properties/
  common.yaml                    Shared configuration
  properties-local.yaml          Local file path and encrypted API token
src/test/munit/
  sales-digest-test-suite.xml     MUnit tests
src/test/resources/test-data/    Deterministic CSV and API fixtures
```

## Deployment approach

For CloudHub 2.0, package the application with `mvn clean package`, select Mule Runtime 4.9.11, and deploy the generated Mule application JAR. Externalize `mule.env`, `encryption.key`, and environment-specific values as secure deployment properties. Provide a file location accessible to the deployed runtime; a developer workstation path is not valid in CloudHub.

For higher sales volume, the next design step would be to replace the local file handoff with durable object storage or messaging, tune `maxConcurrency`, and evaluate batch-level reference-data prefetching. No additional infrastructure is required for this exercise.
