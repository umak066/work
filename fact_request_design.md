# Fact Request Table - Business & Technical Design Document

## Initial Analysis and Data Model Documentation

### Summary

This document presents the initial analysis and proposed data model design for transforming raw OpenTelemetry traces into structured analytical tables. The objective is to enable stakeholders to perform efficient analysis on service performance, latency metrics, and service dependencies for the Cost Estimator application.

---

## 1. Business Context

### 1.1 Purpose

The `fact_request` and `fact_response` tables serve as the primary fact tables for analyzing cost estimation requests. It enables business users and analysts to:

- Analyze request patterns by service code
- Understand provider utilization and distribution
- Track service-provider relationships within requests
- Monitor request volumes and trends over time
- Support operational reporting and business intelligence

### 1.2 Business Requirements

**Analytics Capabilities Required:**

- Requests by service code (e.g., "How many requests for service code Q4019?")
- Requests by provider (e.g., "Which providers are most frequently requested?")
- Service-provider relationship analysis (e.g., "Which providers are requested for service code X?")

**Data Relationships:**

- Each request can contain multiple services (Bundle request)
- Each request can contain up to 5 providers
- Must track the specific relationship between each service and each provider within a request

### 1.3 Source Data

- **Source System:** BigQuery (BQ)
- **Trace Format:** JSON containing request and response spans
- **Extraction Point:** Request body and trace metadata from OpenTelemetry spans
- **Update Frequency:** To be determined (batch/streaming)

---

## 2. Technical Design

### 2.1 Table Grain – fact_request

**One row per request-service-provider combination**

**Example:**
- Request with 1 service and 3 providers = 3 rows
- Request with 2 services and 5 providers = 10 rows

**This grain ensures:**
- Service-provider relationships are explicitly captured
- Analytics by service code or provider are straightforward
- No complex JSON parsing required for common queries

### 2.2 Table Structure

#### Primary Key

**Composite Key:** `global_transaction_id` + `provider_identification_number`

**Note:** If `service_location` is not available or not unique, use `provider_identification_number` as alternative identifier

#### Partitioning Strategy

**Status:** Yet to be decided

**Considerations:**
- Partition by `request_timestamp` (monthly or daily, depending on volume) - potential option
- Benefits if implemented:
  - Improved query performance for time-based queries
  - Easier data archival and retention management
  - Better maintenance operations

---

## 3. Data Model Schema - fact_request

### 3.1 Request-Level Attributes

These attributes are repeated for each service-provider combination within the same request.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `global_transaction_id` | STRING | YES | Global transaction identifier | `x-global-transaction-id` header |
| `trace_id` | STRING | NO | Unique identifier for the trace | `traceId` from span |
| `span_id` | STRING | NO | Unique identifier for the span | `spanId` from span |
| `request_timestamp` | TIMESTAMP | NO | Request start time | `startTimeUnixNano` converted |
| `response_timestamp` | TIMESTAMP | YES | Response end time (if available) | `endTimeUnixNano` converted |
| `request_duration_ms` | FLOAT | YES | Request duration in milliseconds | Calculated from timestamps |
| `api_transaction_id` | STRING | YES | API transaction identifier | `x-apitransactionid` header |
| `http_status_code` | INTEGER | YES | HTTP response status code | `http.status_code` attribute |
| `zip_code` | STRING | YES | Zip code from request | Request body |
| `benefit_product_type` | STRING | YES | Benefit product type (e.g., "Medical") | Request body |
| `language_code` | STRING | YES | Language code | Request body |

### 3.2 Service-Level Attributes

Attributes specific to the service being requested.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `service_code` | STRING | NO | Service code (e.g., "Q4019", "99214") | Request body `service.code` |
| `service_type` | STRING | YES | Service type (e.g., "HCPC", "CPT4") | Request body `service.type` |
| `service_description` | STRING | YES | Service description | Request body `service.description` |
| `supporting_service_code` | STRING | YES | Supporting service code | Request body `service.supportingService.code` |
| `supporting_service_type` | STRING | YES | Supporting service type | Request body `service.supportingService.type` |
| `modifier_code` | STRING | YES | Modifier code | Request body `service.modifier.modifierCode` |
| `diagnosis_code` | STRING | YES | Diagnosis code | Request body `service.diagnosisCode` |
| `place_of_service_code` | STRING | YES | Place of service code | Request body `service.placeOfService.code` |

### 3.3 Provider-Level Attributes

Attributes specific to each provider in the request.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `service_location` | STRING | NO | Service location identifier (part of primary key) | Request body `providerInfo[].serviceLocation` |
| `provider_type` | STRING | YES | Provider type (e.g., "PH", "NP", "MPG") | Request body `providerInfo[].providerType` |
| `speciality_code` | STRING | YES | Provider specialty code | Request body `providerInfo[].speciality.code` |
| `tax_identification_number` | STRING | YES | Tax identification number | Request body `providerInfo[].taxIdentificationNumber` |
| `tax_id_qualifier` | STRING | YES | Tax ID qualifier | Request body `providerInfo[].taxIdQualifier` |
| `network_id` | STRING | YES | Provider network ID | Request body `providerInfo[].providerNetworks.networkID` |
| `provider_identification_number` | STRING | YES | Provider identification number | Request body `providerInfo[].providerIdentificationNumber` |
| `national_provider_id` | STRING | YES | National Provider Identifier (NPI) | Request body `providerInfo[].nationalProviderId` |
| `provider_tier` | STRING | YES | Provider tier | Request body `providerInfo[].providerNetworkParticipation.providerTier` |

### 3.4 Metadata Columns

Operational and audit columns.

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| `created_at` | TIMESTAMP | NO | Record creation timestamp (load time) |
| `updated_at` | TIMESTAMP | YES | Record last update timestamp |

---

## 4. Data Quality & Validation

### 4.1 Required Fields

- `trace_id`
- `span_id`
- `service_code`
- `service_location`
- `request_timestamp`
- `created_at`

### 4.2 Data Validation Rules

- **Service Code:** Cannot be empty or null
- **Service Location:** Must be present and non-empty (required for primary key)
- **Trace ID:** Must be unique within the trace context
- **Timestamp Validation:** `request_timestamp` must be before `response_timestamp` (if both present)
- **Duration Validation:** `request_duration_ms` must be non-negative (if present)

### 4.3 Handling Edge Cases

- **Empty Provider Info:** If provider array is empty, no rows should be created for that request
- **Missing Service Info:** If service object is missing, request should be rejected or flagged
- **Multiple Services:** If request contains multiple services (array), create separate rows for each service-provider combination
- **Duplicate Providers:** If same provider (same `service_location`) appears multiple times for the same service, only one row should be created (deduplication based on primary key constraint)

---

## 5. ETL Considerations

### 5.1 Extraction Logic

1. Parse OpenTelemetry trace JSON from BigQuery
2. Extract request body from span attributes (`request.body`)
3. Parse JSON request body to extract:
   - Request-level attributes
   - Service information (handle single service or array of services)
   - Provider array (up to 5 providers)
4. Generate rows: For each service × each provider combination

### 5.2 Transformation Logic

1. Convert Unix nanoseconds to timestamp format
2. Calculate `request_duration_ms` from start/end timestamps
3. Extract nested JSON fields (e.g., `service.supportingService.code`)
4. Handle empty strings and convert to NULL where appropriate
5. Validate required fields before load

### 5.3 Load Strategy

- **Initial Load:** Full load from historical traces
- **Incremental Load:** Append new requests based on `request_timestamp` or `trace_id`
- **Deduplication:** Use primary key constraint to prevent duplicates

---

## 6. Future Considerations

- Consider for Bundle requests (multiple services in single request)
- Monitor table growth and adjust partitioning strategy if needed
- Consider archiving old partitions based on retention policy
- Evaluate need for materialized views for common aggregations

---

## 7. Dependencies

- Access to BigQuery trace data
- ETL pipeline capable of parsing OpenTelemetry JSON format
- Target database (BigQuery/Spanner) with appropriate capacity
- `fact_response` table (to be designed separately) - will link via `trace_id` + `service_code` + provider identifier

---

## 8. Example Data

### Sample Request

```json
{
  "membershipId": "5~266527209+19+2+20250101+853516+LB+314",
  "benefitProductType": "Medical",
  "service": {
    "code": "S5522",
    "type": "HCPC",
    "description": "",
    "supportingService": {"code": "", "type": ""},
    "modifier": {"modifierCode": ""},
    "diagnosisCode": "",
    "placeOfService": {"code": "12"}
  },
  "providerInfo": [
    {"serviceLocation": "0813535", "providerType": ""},
    {"serviceLocation": "0813535", "providerType": ""},
    {"serviceLocation": "7655903", "providerType": ""}
  ]
}
```

### Resulting Rows in fact_request

- **Row 1:** `global_transaction_id` + "S5522" + "0813535" + provider attributes
- **Row 2:** `global_transaction_id` + "S5522" + "0813535" (duplicate - will be deduplicated by primary key)
- **Row 3:** `global_transaction_id` + "S5522" + "7655903" + provider attributes

---

## 9. Glossary

- **Grain:** The level of detail at which a fact table stores data
- **Trace ID:** Unique identifier for a distributed trace
- **Span ID:** Unique identifier for a single operation within a trace
- **Service Code:** Medical procedure or service code (e.g., CPT4, HCPC codes)
- **Service Location:** Identifier for the provider's service location
- **Bundle Request:** Request containing multiple services
- **Global Transaction ID:** Unique identifier for the entire transaction across systems

---

## Document Information

**Version:** 1.0  
**Last Updated:** [Date]  
**Author:** Data Engineering Team  
**Status:** Draft for Review
