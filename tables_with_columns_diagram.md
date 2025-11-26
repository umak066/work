# BigQuery Tables - Complete Column Listing

## Visual Table Diagram with All Columns

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     cost_estimator_requests                              │
│                        (Main Fact Table)                                 │
├──────────────────────────────────────────────────────────────────────────┤
│ 🔑 request_id                    STRING      PK (DERIVED)                │
│    trace_id                      STRING                                  │
│    span_id                       STRING                                  │
│    global_transaction_id         STRING                                  │
│    api_transaction_id            STRING                                  │
│                                                                           │
│ ⏱️  request_time                  TIMESTAMP                               │
│    response_time                 TIMESTAMP                               │
│    total_duration_ms             FLOAT64                                 │
│                                                                           │
│ 👤 membership_id                  STRING                                  │
│    account_identifier            STRING                                  │
│    user_level_of_assurance       STRING                                  │
│                                                                           │
│ 🏥 service_code                   STRING                                  │
│    service_type                  STRING                                  │
│    service_description           STRING                                  │
│    benefit_product_type          STRING                                  │
│                                                                           │
│ 📊 num_providers_requested        INT64                                   │
│    num_estimates_returned        INT64                                   │
│                                                                           │
│ ✅ http_status_code               INT64                                   │
│    status_code                   INT64                                   │
│    success                       BOOL                                    │
│    error_message                 STRING                                  │
│                                                                           │
│ 📄 response_truncated             BOOL                                    │
│    response_type                 STRING                                  │
│                                                                           │
│ 🔗 oauth_duration_ms              FLOAT64                                 │
│    benefits_api_call_count       INT64                                   │
│    benefits_api_total_duration_ms FLOAT64                                │
│    accumulator_api_duration_ms   FLOAT64                                 │
│                                                                           │
│ 🖥️  service_name                  STRING                                  │
│    deployment_environment        STRING                                  │
│    k8s_cluster_name              STRING                                  │
│    k8s_pod_ip                    STRING                                  │
│    cloud_availability_zone       STRING                                  │
│                                                                           │
│ 💾 request_body_json              STRING      (Full JSON)                 │
│    response_body_json            STRING      (Full JSON)                 │
│                                                                           │
│ 👥 user_roles                     ARRAY<STRING>                           │
│    application_id                STRING                                  │
│                                                                           │
│ 🌐 http_method                    STRING                                  │
│    http_route                    STRING                                  │
│    http_url                      STRING                                  │
│    user_agent                    STRING                                  │
│                                                                           │
│ 📅 ingestion_timestamp            TIMESTAMP                               │
│    request_date                  DATE        PARTITION KEY               │
│                                                                           │
│ PARTITION BY: request_date (Daily)                                       │
│ CLUSTER BY: membership_id, service_code, request_time,                   │
│             deployment_environment                                       │
└──────────────────────────────────────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
┌─────────────────────────────┐  ┌──────────────────────────────┐  ┌─────────────────────────┐
│   requested_providers       │  │  provider_cost_estimates     │  │   request_errors        │
│   (Provider Requests)       │  │  (Cost Estimates)            │  │   (Error Tracking)      │
├─────────────────────────────┤  ├──────────────────────────────┤  ├─────────────────────────┤
│ 🔑 request_id     STRING PK,FK│  │ 🔑 request_id    STRING PK,FK│  │ 🔑 error_id    STRING PK│
│    provider_index  INT64  PK │  │    estimate_index INT64  PK  │  │    request_id  STRING FK│
│                              │  │                              │  │    trace_id    STRING FK│
│ 🔗 trace_id        STRING    │  │ 🔗 trace_id       STRING     │  │    span_id     STRING   │
│    global_transaction_id     │  │                              │  │                         │
│                 STRING       │  │ 🏥 provider_identification_  │  │ ⏱️  error_time  TIMESTAMP│
│                              │  │    number         STRING     │  │    request_duration_    │
│ 🏥 provider_identification_  │  │    national_provider_id      │  │    before_error_ms      │
│    number      STRING        │  │                   STRING     │  │                 FLOAT64 │
│    national_provider_id      │  │    service_location STRING   │  │                         │
│                STRING        │  │    provider_type    STRING   │  │ ⚠️  error_type   STRING │
│    service_location STRING   │  │    tax_id          STRING    │  │    error_severity       │
│    provider_type   STRING    │  │    tax_id_qualifier STRING   │  │                 STRING  │
│                              │  │    speciality_code  STRING   │  │                         │
│ 🏥 speciality_code STRING    │  │                              │  │ 🔴 http_status_code     │
│    network_id      STRING    │  │ 🌐 network_id      STRING    │  │                 INT64   │
│    tax_id          STRING    │  │    provider_tier   STRING    │  │    status_code  INT64   │
│    tax_id_qualifier STRING   │  │                              │  │    error_message STRING │
│                              │  │ ⚠️  has_exception   BOOL      │  │    error_span_name      │
│ 👤 membership_id   STRING    │  │    exception_type   STRING   │  │                 STRING  │
│    service_code    STRING    │  │    exception_title  STRING   │  │    failed_operation     │
│    benefit_product_type      │  │    exception_status INT64    │  │                 STRING  │
│                 STRING       │  │    exception_detail STRING   │  │                         │
│                              │  │    exception_message STRING  │  │ 👤 membership_id STRING │
│ ⏱️  request_time   TIMESTAMP │  │    exception_correlation_id  │  │    service_code STRING  │
│                              │  │                   STRING     │  │    benefit_product_type │
│ 🖥️  deployment_environment   │  │                              │  │                 STRING  │
│                 STRING       │  │ ✅ is_service_covered STRING │  │    num_providers_       │
│    request_date    DATE      │  │    max_coverage_amount       │  │    requested    INT64   │
│                 PARTITION    │  │                   FLOAT64    │  │                         │
│                              │  │                              │  │ 🌐 http_method  STRING  │
│ PARTITION BY: request_date   │  │ 💰 in_network_cost  FLOAT64  │  │    http_route   STRING  │
│ CLUSTER BY:                  │  │    out_of_network_cost       │  │    http_url     STRING  │
│   provider_identification_   │  │                   FLOAT64    │  │                         │
│   number, service_location,  │  │    in_network_cost_type      │  │ 🖥️  service_name STRING │
│   membership_id, request_time│  │                   STRING     │  │    deployment_          │
└─────────────────────────────┘  │                              │  │    environment  STRING  │
                                  │ 💵 copay_amount     FLOAT64  │  │    k8s_cluster_name     │
                                  │    coinsurance_amount        │  │                 STRING  │
                                  │                   FLOAT64    │  │    k8s_pod_ip   STRING  │
                                  │    coinsurance_percent       │  │                         │
                                  │                   FLOAT64    │  │ 🔗 external_api_url     │
                                  │    member_responsibility_    │  │                 STRING  │
                                  │    amount         FLOAT64    │  │    external_api_status_ │
                                  │    member_responsibility_    │  │    code         INT64   │
                                  │    percent        FLOAT64    │  │                         │
                                  │    amount_payable FLOAT64    │  │ 💾 request_body_json    │
                                  │                              │  │                 STRING  │
                                  │ 📊 DEDUCTIBLE - INDIVIDUAL:  │  │                         │
                                  │    deductible_individual_    │  │ 📅 ingestion_timestamp  │
                                  │    limit          FLOAT64    │  │                TIMESTAMP│
                                  │    deductible_individual_    │  │    error_date   DATE    │
                                  │    current        FLOAT64    │  │                PARTITION│
                                  │    deductible_individual_    │  │                         │
                                  │    calculated     FLOAT64    │  │ PARTITION BY: error_date│
                                  │    deductible_individual_    │  │ CLUSTER BY:             │
                                  │    remaining      FLOAT64    │  │   error_type,           │
                                  │    deductible_individual_    │  │   deployment_environment│
                                  │    applied        FLOAT64    │  │   error_time,           │
                                  │                              │  │   membership_id         │
                                  │ 📊 DEDUCTIBLE - FAMILY:      │  └─────────────────────────┘
                                  │    deductible_family_limit   │
                                  │                   FLOAT64    │
                                  │    deductible_family_current │
                                  │                   FLOAT64    │
                                  │    deductible_family_        │
                                  │    calculated     FLOAT64    │
                                  │    deductible_family_        │
                                  │    remaining      FLOAT64    │
                                  │    deductible_family_applied │
                                  │                   FLOAT64    │
                                  │                              │
                                  │ 📊 OOP MAX - INDIVIDUAL:     │
                                  │    oop_individual_limit      │
                                  │                   FLOAT64    │
                                  │    oop_individual_current    │
                                  │                   FLOAT64    │
                                  │    oop_individual_calculated │
                                  │                   FLOAT64    │
                                  │    oop_individual_remaining  │
                                  │                   FLOAT64    │
                                  │    oop_individual_applied    │
                                  │                   FLOAT64    │
                                  │                              │
                                  │ 📊 OOP MAX - FAMILY:         │
                                  │    oop_family_limit FLOAT64  │
                                  │    oop_family_current        │
                                  │                   FLOAT64    │
                                  │    oop_family_calculated     │
                                  │                   FLOAT64    │
                                  │    oop_family_remaining      │
                                  │                   FLOAT64    │
                                  │    oop_family_applied        │
                                  │                   FLOAT64    │
                                  │                              │
                                  │ 👤 membership_id   STRING    │
                                  │    service_code    STRING    │
                                  │    service_type    STRING    │
                                  │    benefit_product_type      │
                                  │                   STRING     │
                                  │                              │
                                  │ ⏱️  request_time   TIMESTAMP │
                                  │                              │
                                  │ 🖥️  deployment_environment   │
                                  │                   STRING     │
                                  │    request_date    DATE      │
                                  │                 PARTITION    │
                                  │                              │
                                  │ PARTITION BY: request_date   │
                                  │ CLUSTER BY:                  │
                                  │   provider_identification_   │
                                  │   number, service_location,  │
                                  │   membership_id, request_time│
                                  └──────────────────────────────┘
```

---

## Table Summary

| Table | Primary Key | Partition | Clustering | Est. Rows/Month |
|-------|-------------|-----------|------------|-----------------|
| `cost_estimator_requests` | `request_id` (DERIVED) | `request_date` | `membership_id`, `service_code`, `request_time`, `deployment_environment` | 1M-10M |
| `requested_providers` | `(request_id, provider_index)` | `request_date` | `provider_identification_number`, `service_location`, `membership_id`, `request_time` | 3M-50M |
| `provider_cost_estimates` | `(request_id, estimate_index)` | `request_date` | `provider_identification_number`, `service_location`, `membership_id`, `request_time` | 2M-40M |
| `request_errors` | `error_id` (DERIVED) | `error_date` | `error_type`, `deployment_environment`, `error_time`, `membership_id` | 100K-1M |

---

## Column Count Summary

| Table | Total Columns | Key Columns | Business Columns | Metadata Columns |
|-------|---------------|-------------|------------------|------------------|
| `cost_estimator_requests` | 35 | 3 (IDs) | 25 | 7 |
| `requested_providers` | 16 | 2 (composite PK) | 10 | 4 |
| `provider_cost_estimates` | 52 | 2 (composite PK) | 43 (incl 20 accumulator fields) | 7 |
| `request_errors` | 23 | 3 (IDs) | 14 | 6 |

**Total: 126 columns across 4 tables**

---

## Accumulator Field Groups (20 fields total in `provider_cost_estimates`)

### Deductible - Individual (5 fields)
- `deductible_individual_limit`
- `deductible_individual_current`
- `deductible_individual_calculated`
- `deductible_individual_remaining`
- `deductible_individual_applied`

### Deductible - Family (5 fields)
- `deductible_family_limit`
- `deductible_family_current`
- `deductible_family_calculated`
- `deductible_family_remaining`
- `deductible_family_applied`

### OOP Max - Individual (5 fields)
- `oop_individual_limit`
- `oop_individual_current`
- `oop_individual_calculated`
- `oop_individual_remaining`
- `oop_individual_applied`

### OOP Max - Family (5 fields)
- `oop_family_limit`
- `oop_family_current`
- `oop_family_calculated`
- `oop_family_remaining`
- `oop_family_applied`

---

## Relationships

```
cost_estimator_requests (1) ──→ (N) requested_providers
                         (1) ──→ (N) provider_cost_estimates
                         (1) ──→ (M) request_errors

requested_providers (1) ──→ (0..1) provider_cost_estimates
```

**Join Keys:**
- Request → Providers: `request_id`
- Request → Estimates: `request_id`
- Request → Errors: `request_id`
- Providers ↔ Estimates: `request_id` + `provider_identification_number`
