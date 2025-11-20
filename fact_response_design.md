# Fact Response Table - Business & Technical Design Document

## Initial Analysis and Data Model Documentation

### Summary

This document presents the design for the `fact_response` table, which stores cost estimation response data extracted from OpenTelemetry traces. The table is designed to capture response data from the `costEstimateResponseInfo` array, enabling analytics on coverage, costs, and health claim line information while maintaining the relationship with corresponding requests.

**Key Design Principle:** This table is normalized - it stores only response-specific data (coverage, costs, exceptions) and join keys. Service-level and provider-level descriptive attributes are NOT stored here - they must be joined from `fact_request` table using the join keys (`trace_id`, `service_code`, `service_location`).

---

## 1. Business Context

### 1.1 Purpose

The `fact_response` table serves as the primary fact table for analyzing cost estimation responses. It enables business users and analysts to:

- Analyze cost estimates by service code and provider
- Understand coverage patterns and cost sharing
- Track health claim line details (copay, coinsurance, responsibility amounts)
- Monitor and analyze exceptions (error types, frequencies, patterns)
- Monitor response data quality and completeness
- Support operational reporting and business intelligence
- Link response data to corresponding requests for end-to-end analysis

### 1.2 Business Requirements

**Analytics Capabilities Required:**

- Response analysis by service code
- Response analysis by provider
- Cost analysis (in-network vs out-of-network costs)
- Coverage analysis (service coverage rates, cost sharing patterns)
- Health claim line analysis (copay, coinsurance, responsibility amounts)
- Exception analysis (exception types, frequencies, patterns by service code/provider)

**Data Relationships:**

- Each response corresponds to a request (linked via `trace_id` + `service_code` + `service_location`)
- Each response can contain multiple provider responses (from `costEstimateResponseInfo` array)
- Response structure may have ambiguity (response providers may not match request providers exactly)

### 1.3 Source Data

- **Source System:** BigQuery (BQ)
- **Trace Format:** JSON containing request and response spans
- **Extraction Point:** Response body from OpenTelemetry span attributes (`response.body`)
- **Update Frequency:** To be determined (batch/streaming)

---

## 2. Technical Design

### 2.1 Table Grain – fact_response

**One row per response-service-provider combination**

Each element in the `costEstimateResponseInfo` array becomes one row.

**Example:**
- Response with 1 service and 3 providers in `costEstimateResponseInfo` = 3 rows
- Response with 1 service and 1 provider = 1 row

**This grain ensures:**
- Response-provider relationships are explicitly captured
- Analytics by service code or provider are straightforward
- Natural alignment with `fact_request` table for joining
- No complex JSON parsing required for common queries

### 2.2 Table Structure

#### Primary Key

**Composite Key:** `trace_id` + `service_code` + `service_location`

**Note:** 
- `service_code` and `service_location` are join keys to fact_request (not descriptive attributes)
- The `service_location` comes from `costEstimateResponseInfo[].providerInfo.serviceLocation` in the response
- This may differ from the request `service_location`, so linking should be flexible
- Service-level and provider-level descriptive attributes are NOT stored in fact_response - they must be joined from fact_request

#### Linking to fact_request

**Primary Link Strategy:**
- Join via: `fact_request.trace_id = fact_response.trace_id` AND `fact_request.service_code = fact_response.service_code` AND `fact_request.service_location = fact_response.service_location`
- If `service_location` doesn't match, use `provider_identification_number` as fallback join key
- After joining, all service-level and provider-level attributes are available from fact_request
- Consider adding a `matches_request_provider` flag to indicate successful matching (optional)

#### Partitioning Strategy

**Status:** Yet to be decided

**Considerations:**
- Partition by `response_timestamp` (monthly or daily, depending on volume) - potential option
- Benefits if implemented:
  - Improved query performance for time-based queries
  - Easier data archival and retention management
  - Better maintenance operations

---

## 3. Data Model Schema - fact_response

### 3.1 Response-Level Attributes

These attributes are repeated for each service-provider combination within the same response.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `trace_id` | STRING | NO | Unique identifier for the trace (join key to fact_request) | `traceId` from span |
| `span_id` | STRING | NO | Unique identifier for the span | `spanId` from span |
| `response_timestamp` | TIMESTAMP | NO | Response end time | `endTimeUnixNano` converted |
| `http_status_code` | INTEGER | YES | HTTP response status code | `http.status_code` attribute |
| `raw_response_json` | JSON/STRING | YES | Full response JSON for debugging | `response.body` attribute (full JSON) |

### 3.2 Join Keys to fact_request

**Note:** Service-level and provider-level attributes are NOT stored in fact_response. They must be joined from fact_request using these keys.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `service_code` | STRING | NO | Service code (join key to fact_request) | `costEstimateResponse.service.code` |
| `service_location` | STRING | NO | Service location identifier (join key to fact_request, part of primary key) | `costEstimateResponseInfo[].providerInfo.serviceLocation` |
| `provider_identification_number` | STRING | YES | Provider identification number (fallback join key to fact_request) | `costEstimateResponseInfo[].providerInfo.providerIdentificationNumber` |

**Join Strategy:**
- **Primary Join:** `fact_request.trace_id = fact_response.trace_id` AND `fact_request.service_code = fact_response.service_code` AND `fact_request.service_location = fact_response.service_location`
- **Fallback Join:** If service_location doesn't match, use `fact_request.provider_identification_number = fact_response.provider_identification_number`

**Service-Level Attributes Available in fact_request:**
- `service_type`, `service_description`, `supporting_service_code`, `supporting_service_type`, `modifier_code`, `diagnosis_code`, `place_of_service_code`

**Provider-Level Attributes Available in fact_request:**
- `provider_type`, `speciality_code`, `tax_identification_number`, `tax_id_qualifier`, `network_id`, `national_provider_id`, `provider_tier`

### 3.3 Coverage Measures

Coverage information from `costEstimateResponseInfo[].coverage`.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `is_service_covered` | STRING | YES | Is service covered (Y/N) | `costEstimateResponseInfo[].coverage.isServiceCovered` |
| `max_coverage_amount` | FLOAT | YES | Maximum coverage amount | `costEstimateResponseInfo[].coverage.maxCoverageAmount` |
| `cost_share_copay` | FLOAT | YES | Cost share copay amount | `costEstimateResponseInfo[].coverage.costShareCopay` |
| `cost_share_coinsurance` | INTEGER | YES | Cost share coinsurance percentage | `costEstimateResponseInfo[].coverage.costShareCoinsurance` |

### 3.4 Cost Measures

Cost information from `costEstimateResponseInfo[].cost`.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `in_network_costs` | FLOAT | YES | In-network costs | `costEstimateResponseInfo[].cost.inNetworkCosts` |
| `out_of_network_costs` | FLOAT | YES | Out-of-network costs | `costEstimateResponseInfo[].cost.outOfNetworkCosts` |
| `in_network_costs_type` | STRING | YES | In-network costs type (AMOUNT/PERCENTAGE) | `costEstimateResponseInfo[].cost.inNetworkCostsType` |

### 3.5 HealthClaimLine Measures

Health claim line information from `costEstimateResponseInfo[].healthClaimLine`.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `amount_copay` | FLOAT | YES | Amount copay | `costEstimateResponseInfo[].healthClaimLine.amountCopay` |
| `amount_coinsurance` | FLOAT | YES | Amount coinsurance | `costEstimateResponseInfo[].healthClaimLine.amountCoinsurance` |
| `amount_responsibility` | FLOAT | YES | Amount responsibility | `costEstimateResponseInfo[].healthClaimLine.amountResponsibility` |
| `percent_responsibility` | FLOAT | YES | Percent responsibility | `costEstimateResponseInfo[].healthClaimLine.percentResponsibility` |
| `amount_payable` | FLOAT | YES | Amount payable | `costEstimateResponseInfo[].healthClaimLine.amountpayable` |

### 3.6 Accumulators

Accumulator information from `costEstimateResponseInfo[].accumulators`.

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `accumulators_json` | JSON/STRING | YES | Accumulators as JSON array | `costEstimateResponseInfo[].accumulators` (entire array) |

### 3.7 Exception Attributes

Exception information from `costEstimateResponseInfo[].exception`. Exceptions occur at the provider level and are mutually exclusive with coverage/cost data (if exception exists, coverage/cost fields will be NULL).

| Column Name | Data Type | Nullable | Description | Source |
|------------|-----------|----------|-------------|--------|
| `has_exception` | BOOLEAN | NO | Indicates if this provider response has an exception | Presence of `costEstimateResponseInfo[].exception` object |
| `exception_correlation_id` | STRING | YES | Correlation ID for the exception | `costEstimateResponseInfo[].exception.correlationId` |
| `exception_type` | STRING | YES | Exception type (RFC URL or error type) | `costEstimateResponseInfo[].exception.type` |
| `exception_title` | STRING | YES | Exception title (e.g., "Rate not found", "Provider not found") | `costEstimateResponseInfo[].exception.title` |
| `exception_status` | INTEGER | YES | Exception HTTP status code (e.g., 500) | `costEstimateResponseInfo[].exception.status` |
| `exception_detail` | STRING | YES | Detailed exception message | `costEstimateResponseInfo[].exception.detail` |
| `exception_message` | STRING | YES | Exception message | `costEstimateResponseInfo[].exception.message` |

**Note:** When `has_exception = true`, the coverage, cost, and healthClaimLine fields will typically be NULL as exceptions replace normal response data at the provider level.

### 3.8 Metadata Columns

Operational and audit columns.

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| `created_at` | TIMESTAMP | NO | Record creation timestamp (load time) |
| `updated_at` | TIMESTAMP | YES | Record last update timestamp |

---

## 4. Data Quality & Validation

### 4.1 Required Fields

- `trace_id` (join key to fact_request)
- `span_id`
- `service_code` (join key to fact_request)
- `service_location` (join key to fact_request)
- `response_timestamp`
- `created_at`

### 4.2 Data Validation Rules

- **Service Code:** Cannot be empty or null (required for joining to fact_request)
- **Service Location:** Must be present and non-empty (required for primary key and joining to fact_request)
- **Trace ID:** Must be unique within the trace context (required for joining to fact_request)
- **Response Timestamp:** Must be valid timestamp
- **Numeric Fields:** Cost and amount fields should be non-negative (if present)
- **Join Key Validation:** Ensure join keys (trace_id, service_code, service_location) exist in fact_request for successful joins

### 4.3 Handling Edge Cases

- **Empty costEstimateResponseInfo Array:** If array is empty, no rows should be created for that response
- **Missing Provider Info:** If providerInfo is missing in an array element, row should be rejected or flagged
- **Response Ambiguity:** Response providers may not match request providers - store all response providers regardless of match status
- **Missing Fields:** All response fields should be nullable to handle missing data gracefully
- **Orphan Responses:** Response providers that don't match any request provider should still be stored (for analysis)
- **Truncated Responses:** If response is incomplete, store available data and flag if needed
- **Provider-Level Exceptions:** If a provider has an exception, store exception data and set coverage/cost fields to NULL (exceptions are mutually exclusive with normal response data)
- **Mixed Responses:** A single response can have some providers with exceptions and others with normal data - each provider becomes a separate row

### 4.4 Response Ambiguity Handling

**Key Ambiguity Scenarios:**

1. **Provider Mismatch:** Response `service_location` doesn't match any request `service_location`
   - **Action:** Store response row with join keys, use `provider_identification_number` for alternative linking
   - **Impact:** Orphan response - service/provider attributes from fact_request will not be available via join

2. **Missing Response:** Request exists but no response found
   - **Action:** No row in fact_response (left join from fact_request will show NULL response data)

3. **Extra Providers:** Response has more providers than request
   - **Action:** Store all response providers (orphan rows) - these won't have matching service/provider attributes from fact_request

4. **Fewer Providers:** Response has fewer providers than request
   - **Action:** Only store providers that exist in response - matching requests will have NULL response data

5. **Field Variations:** Response structure differs from expected
   - **Action:** Store available response fields, NULL for missing fields, preserve in `raw_response_json`
   - **Note:** Service/provider attributes are not stored in fact_response regardless - always join from fact_request

6. **Join Key Mismatch:** Response join keys don't match any fact_request record
   - **Action:** Store response row anyway (orphan response)
   - **Impact:** Cannot retrieve service/provider attributes - use raw_response_json for debugging

---

## 5. ETL Considerations

### 5.1 Extraction Logic

1. Parse OpenTelemetry trace JSON from BigQuery
2. Extract response body from span attributes (`response.body`)
3. Parse JSON response body to extract:
   - Response-level attributes
   - **Join keys only:** `service_code` from `costEstimateResponse.service.code` and `service_location` from `costEstimateResponseInfo[].providerInfo.serviceLocation`
   - `costEstimateResponseInfo` array
4. For each element in `costEstimateResponseInfo` array:
   - Extract join keys: `service_location` and `provider_identification_number` (for fallback join)
   - **Do NOT extract** service-level or provider-level descriptive attributes (get from fact_request via join)
   - Check if `exception` object exists
   - If exception exists: Extract exception fields, set coverage/cost fields to NULL
   - If exception doesn't exist: Extract coverage, cost, healthClaimLine, accumulators
   - Generate one row per array element

### 5.2 Transformation Logic

1. Convert Unix nanoseconds to timestamp format for `response_timestamp`
2. **Extract join keys only:** `service_code` and `service_location` (do not extract other service/provider attributes)
3. Handle empty strings and convert to NULL where appropriate
4. **Exception Handling:**
   - If `exception` object exists in `costEstimateResponseInfo[]` element:
     - Set `has_exception = true`
     - Extract all exception fields
     - Set coverage, cost, and healthClaimLine fields to NULL
   - If `exception` object doesn't exist:
     - Set `has_exception = false`
     - Extract coverage, cost, healthClaimLine fields
     - Set exception fields to NULL
5. Store `accumulators` array as JSON string/JSON type
6. Store full `response.body` as `raw_response_json` for debugging
7. Validate required fields (especially join keys) before load
8. Handle missing or malformed response structures gracefully
9. **Join Key Validation:** Verify that join keys (trace_id, service_code, service_location) can be matched to fact_request records

### 5.3 Load Strategy

- **Initial Load:** Full load from historical traces
- **Incremental Load:** Append new responses based on `response_timestamp` or `trace_id`
- **Deduplication:** Use primary key constraint to prevent duplicates
- **Linking:** After load, perform analysis to identify response-request matches and mismatches

---

## 6. Linking Strategy with fact_request

### 6.1 Primary Join Strategy

**Join Condition:**
```sql
SELECT 
    fr.*,  -- fact_response columns
    fq.service_type,  -- Service-level attributes from fact_request
    fq.service_description,
    fq.supporting_service_code,
    fq.supporting_service_type,
    fq.modifier_code,
    fq.diagnosis_code,
    fq.place_of_service_code,
    fq.provider_type,  -- Provider-level attributes from fact_request
    fq.speciality_code,
    fq.tax_identification_number,
    fq.tax_id_qualifier,
    fq.network_id,
    fq.national_provider_id,
    fq.provider_tier,
    fq.request_timestamp,  -- Request-level attributes
    fq.global_transaction_id,
    fq.api_transaction_id,
    fq.zip_code,
    fq.benefit_product_type,
    fq.language_code
FROM fact_response fr
INNER JOIN fact_request fq
    ON fq.trace_id = fr.trace_id
    AND fq.service_code = fr.service_code
    AND fq.service_location = fr.service_location
```

### 6.2 Fallback Join Strategy

If primary join fails (service_location mismatch), try:
```sql
SELECT 
    fr.*,
    fq.*  -- All fact_request attributes
FROM fact_response fr
LEFT JOIN fact_request fq
    ON fq.trace_id = fr.trace_id
    AND fq.service_code = fr.service_code
    AND fq.service_location = fr.service_location
LEFT JOIN fact_request fq_fallback
    ON fq_fallback.trace_id = fr.trace_id
    AND fq_fallback.service_code = fr.service_code
    AND fq_fallback.provider_identification_number = fr.provider_identification_number
WHERE fq.trace_id IS NULL  -- Primary join failed
    AND fq_fallback.trace_id IS NOT NULL  -- Fallback join succeeded
```

### 6.3 Orphan Response Handling

- Responses that don't match any request should still be stored
- Can be identified via LEFT JOIN analysis:
  ```sql
  SELECT fr.*
  FROM fact_response fr
  LEFT JOIN fact_request fq
      ON fq.trace_id = fr.trace_id
      AND fq.service_code = fr.service_code
      AND fq.service_location = fr.service_location
  WHERE fq.trace_id IS NULL  -- No matching request found
  ```
- Useful for understanding response patterns and data quality issues
- **Note:** Orphan responses will not have service/provider attributes available (since they can't be joined to fact_request)

### 6.4 Benefits of Normalized Design

- **Reduced Storage:** Service and provider attributes stored once in fact_request
- **Data Consistency:** Single source of truth for service/provider information
- **Easier Maintenance:** Updates to service/provider attributes only need to be made in fact_request
- **Clear Separation:** Request data (what was asked) vs Response data (what was returned)

---

## 7. Query Patterns & Performance

### 7.1 Common Query Patterns

**Query 1: Responses by Service Code (with service details from fact_request)**
```sql
SELECT 
    fr.service_code,
    fq.service_type,
    fq.service_description,
    COUNT(*) as response_count
FROM fact_response fr
INNER JOIN fact_request fq
    ON fq.trace_id = fr.trace_id
    AND fq.service_code = fr.service_code
    AND fq.service_location = fr.service_location
WHERE fr.response_timestamp >= '2025-01-01'
GROUP BY fr.service_code, fq.service_type, fq.service_description
ORDER BY response_count DESC;
```

**Query 2: Cost Analysis by Provider (with provider details from fact_request)**
```sql
SELECT 
    fr.service_location,
    fq.provider_identification_number,
    fq.provider_type,
    fq.network_id,
    AVG(fr.in_network_costs) as avg_in_network_cost,
    SUM(fr.amount_payable) as total_payable
FROM fact_response fr
INNER JOIN fact_request fq
    ON fq.trace_id = fr.trace_id
    AND fq.service_code = fr.service_code
    AND fq.service_location = fr.service_location
WHERE fr.response_timestamp >= '2025-01-01'
GROUP BY fr.service_location, fq.provider_identification_number, fq.provider_type, fq.network_id;
```

**Query 3: Join Request and Response (Full Details)**
```sql
SELECT 
    fq.service_code,
    fq.service_type,
    fq.service_location,
    fq.provider_type,
    fq.network_id,
    fq.request_timestamp,
    fr.response_timestamp,
    fr.in_network_costs,
    fr.amount_payable,
    fr.has_exception,
    fr.exception_title
FROM fact_request fq
LEFT JOIN fact_response fr
    ON fq.trace_id = fr.trace_id
    AND fq.service_code = fr.service_code
    AND fq.service_location = fr.service_location
WHERE fq.request_timestamp >= '2025-01-01';
```

**Query 4: Exception Analysis (with service/provider details from fact_request)**
```sql
SELECT 
    fr.service_code,
    fq.service_type,
    fq.provider_type,
    fr.exception_title,
    fr.exception_status,
    COUNT(*) as exception_count,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (PARTITION BY fr.service_code) as exception_percentage
FROM fact_response fr
INNER JOIN fact_request fq
    ON fq.trace_id = fr.trace_id
    AND fq.service_code = fr.service_code
    AND fq.service_location = fr.service_location
WHERE fr.has_exception = true
    AND fr.response_timestamp >= '2025-01-01'
GROUP BY fr.service_code, fq.service_type, fq.provider_type, fr.exception_title, fr.exception_status
ORDER BY exception_count DESC;
```

### 7.2 Performance Optimization

- **Partitioning:** Yet to be decided (see Section 2.2)
- **Clustering:** Cluster by join keys (`trace_id`, `service_code`, `service_location`) for optimal join performance with fact_request
- **Indexes:**
  - Primary index on (`trace_id`, `service_code`, `service_location`) - join keys
  - Index on `trace_id` (for trace-level queries and joining with fact_request)
  - Index on `service_code` (for service-based analytics - note: service details come from fact_request)
  - Index on `service_location` (for provider-based analytics - note: provider details come from fact_request)
  - Index on `response_timestamp` (for time-based queries)
- **Join Optimization:**
  - Ensure fact_request has matching indexes on join keys (`trace_id`, `service_code`, `service_location`)
  - Consider materialized views for frequently joined queries
  - Use INNER JOIN when service/provider details are required (filters out orphan responses)
  - Use LEFT JOIN when analyzing all responses including orphans

---

## 8. Future Considerations

- Consider normalizing accumulators into separate table if query requirements emerge
- Monitor response-request matching rates to identify data quality issues
- Consider adding derived metrics (e.g., cost per service, coverage rates)
- Evaluate need for materialized views for common aggregations
- Consider adding response quality indicators if needed in future

---

## 9. Dependencies

- Access to BigQuery trace data
- ETL pipeline capable of parsing OpenTelemetry JSON format
- Target database (BigQuery/Spanner) with appropriate capacity
- `fact_request` table (must exist for linking)

---

## 10. Example Data

### Sample Response

From trace11.json, the response structure:

```json
{
  "costEstimateResponse": {
    "service": {
      "code": "36836",
      "type": "CPT4",
      "description": "",
      "supportingService": {"code": "", "type": ""},
      "modifier": {"modifierCode": ""},
      "diagnosisCode": "",
      "placeOfService": {"code": "24"}
    },
    "costEstimateResponseInfo": [
      {
        "providerInfo": {
          "serviceLocation": "1995752",
          "providerType": "PH",
          "speciality": {"code": "10301"},
          "taxIdentificationNumber": "",
          "taxIdQualifier": "",
          "providerNetworks": {"networkID": "01344"},
          "providerIdentificationNumber": "0004115105",
          "nationalProviderId": "1194771162",
          "providerNetworkParticipation": {"providerTier": ""}
        },
        "coverage": {
          "isServiceCovered": "Y",
          "maxCoverageAmount": 0,
          "costShareCopay": 0,
          "costShareCoinsurance": 0
        },
        "cost": {
          "inNetworkCosts": 7665.64,
          "outOfNetworkCosts": 0,
          "inNetworkCostsType": "AMOUNT"
        },
        "healthClaimLine": {
          "amountCopay": 0,
          "amountCoinsurance": 0,
          "amountResponsibility": 0,
          "percentResponsibility": 0,
          "amountpayable": 7665.64
        },
        "accumulators": []
      }
    ]
  }
}
```

### Resulting Row in fact_response (Success Case)

**Join Keys (for joining to fact_request):**
- **trace_id:** "2d79403109333e0199cd97a42371124a"
- **service_code:** "36836"
- **service_location:** "1995752"

**Response Data:**
- **has_exception:** false
- **in_network_costs:** 7665.64
- **amount_payable:** 7665.64
- **is_service_covered:** "Y"
- **accumulators_json:** "[]" (empty array)

**Note:** Service-level attributes (service_type, service_description, etc.) and provider-level attributes (provider_type, network_id, etc.) are NOT stored in fact_response. They must be joined from fact_request using the join keys above.

### Sample Response with Exception

From api_sample_responses.json, a response with exception:

```json
{
  "costEstimateResponse": {
    "service": {
      "code": "994",
      "type": "CPT4",
      "description": "",
      "supportingService": {"code": "", "type": ""},
      "modifier": {"modifierCode": ""},
      "diagnosisCode": "",
      "placeOfService": {"code": "11"}
    },
    "costEstimateResponseInfo": [
      {
        "providerInfo": {
          "serviceLocation": "101250",
          "providerType": "PH",
          "speciality": {"code": ""},
          "taxIdentificationNumber": "",
          "taxIdQualifier": "",
          "providerNetworks": {"networkID": "09278"},
          "providerIdentificationNumber": "0004326320",
          "nationalProviderId": "",
          "providerNetworkParticipation": {"providerTier": ""}
        },
        "exception": {
          "correlationId": "int-0tqv2iczhpgub1fnubsd",
          "type": "https://www.rfc-editor.org/rfc/rfc7231#section-6.5.4",
          "title": "Rate not found",
          "status": 500,
          "detail": "Rate not found for the given criteria",
          "message": "Rate not found for the provided criteria"
        }
      }
    ]
  }
}
```

### Resulting Row in fact_response (Exception Case)

**Join Keys (for joining to fact_request):**
- **trace_id:** [trace_id from span]
- **service_code:** "994"
- **service_location:** "101250"

**Exception Data:**
- **has_exception:** true
- **exception_correlation_id:** "int-0tqv2iczhpgub1fnubsd"
- **exception_type:** "https://www.rfc-editor.org/rfc/rfc7231#section-6.5.4"
- **exception_title:** "Rate not found"
- **exception_status:** 500
- **exception_detail:** "Rate not found for the given criteria"
- **exception_message:** "Rate not found for the provided criteria"
- **in_network_costs:** NULL (exception replaces cost data)
- **amount_payable:** NULL (exception replaces cost data)
- **is_service_covered:** NULL (exception replaces coverage data)

**Note:** Service-level attributes (service_type, service_description, etc.) and provider-level attributes (provider_type, network_id, etc.) are NOT stored in fact_response. They must be joined from fact_request using the join keys above.

---

## 11. Glossary

- **Grain:** The level of detail at which a fact table stores data
- **Trace ID:** Unique identifier for a distributed trace
- **Span ID:** Unique identifier for a single operation within a trace
- **Service Code:** Medical procedure or service code (e.g., CPT4, HCPC codes)
- **Service Location:** Identifier for the provider's service location
- **costEstimateResponseInfo:** Array in response containing provider-specific cost estimates
- **Orphan Response:** Response provider that doesn't match any request provider
- **Response Ambiguity:** Situation where response structure doesn't perfectly align with request structure
- **Provider-Level Exception:** Exception that occurs at the provider level within `costEstimateResponseInfo[]`, replacing normal coverage/cost data for that specific provider
- **Exception Correlation ID:** Unique identifier for tracking exceptions across systems

---

## Document Information

**Version:** 1.0  
**Last Updated:** [Date]  
**Author:** Data Engineering Team  
**Status:** Draft for Review

