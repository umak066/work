# OpenTelemetry Trace Data Model Design
## Initial Analysis and Data Model Documentation

**Document Version:** 1.0  
**Date:** January 2025  
**Author:** Senior Data Modeler  
**Project:** Cost Estimator Data Pipelines - Trace Analytics

---

## Executive Summary

This document presents the initial analysis and proposed data model design for transforming raw OpenTelemetry traces into structured analytical tables. The objective is to enable stakeholders to perform efficient analysis on service performance, latency metrics, and service dependencies for the Cost Estimator application.

### Business Objectives

- **Performance Monitoring**: Track service latency, throughput, and response times
- **Error Analysis**: Identify and analyze failures, exceptions, and error patterns
- **Dependency Mapping**: Understand service-to-service call patterns and dependencies
- **Business Context Analysis**: Correlate technical metrics with business entities (memberships, service codes, benefit types)

### Scope

- Source: OpenTelemetry trace data stored as JSON strings in BigQuery
- Target: Structured analytical tables optimized for query performance
- Users: Data analysts, SRE teams, product managers, business stakeholders

---

## 1. Data Source Analysis

### 1.1 Sample Data Overview

Analysis performed on 5 sample trace files (trace1.json through trace5.json) reveals the following structure:

#### Trace Structure
```
batches[]
  └── resource
      └── attributes[] (service metadata)
  └── instrumentationLibrarySpans[]
      └── spans[]
          ├── Basic Info (traceId, spanId, parentSpanId, name, kind)
          ├── Timing (startTimeUnixNano, endTimeUnixNano)
          ├── attributes[] (key-value pairs)
          ├── events[] (exceptions, logs)
          └── status (code, message)
```

### 1.2 Key Data Elements Identified

#### Resource Attributes (Service-Level Metadata)
- `service.name`: APM0016344
- `service.version`: 1.0.0
- `deployment.environment`: hcb-dev
- `cloud.provider`: gcp
- `cloud.account.id`: anbc-hcb-dev
- `cloud.platform`: gcp_kubernetes_engine
- `k8s.cluster.name`: io-hcb-dev
- `k8s.pod.ip`: Various IPs
- `host.name`: gke-io-hcb-dev-default-f-cc51fb65-qqcd

#### Span Attributes (Request/Response Context)
**HTTP Attributes:**
- `http.method`: POST
- `http.url`: Full request URLs
- `http.status_code`: 200, 400, 500
- `http.route`: /costestimator/v1/rate
- `http.path`: /costestimator/v1/rate
- `http.scheme`: http
- `http.host`: IP:port combinations
- `http.user_agent`: axios/1.11.0

**Business Context Attributes:**
- `request.membership_id`: Various membership identifiers
- `request.service_code`: CPT4/HCPC codes (99454, Q4019, 0361U, 73030)
- `request.benefit_product_type`: Medical
- `request.num_providers`: Integer (typically 5)
- `request.body`: Full JSON request payload
- `request.header.*`: Various header values
- `response.body`: Full JSON response payload (when available)

**Function/Operation Attributes:**
- `function.name`: estimate_cost
- `function.module`: app.api.v1.routes.requests
- `function.duration_ms`: Execution duration
- `operation`: cost_estimation_request
- `service.name`: cost-estimator-calc-service

**Error Attributes:**
- `error.type`: ValueError
- `error.message`: Header x-eieusercontext is required

#### Span Events (Exceptions)
- `exception.type`: ValueError
- `exception.message`: Error messages
- `exception.stacktrace`: Full Python stack traces
- `exception.escaped`: Boolean flag

#### Span Types Observed
- **SPAN_KIND_SERVER**: Entry point spans (POST /costestimator/v1/rate)
- **SPAN_KIND_CLIENT**: Outbound HTTP calls to external services
- **SPAN_KIND_INTERNAL**: Internal processing spans

### 1.3 Data Patterns and Observations

#### Trace Patterns
1. **Single Service Traces**: All traces originate from service "APM0016344"
2. **Hierarchical Structure**: Parent-child relationships via parentSpanId
3. **Multiple Batches**: Some traces contain multiple batches with same resource
4. **External Service Calls**: Calls to qaapi01.int.aetna.com for:
   - OAuth token retrieval
   - Services/benefits details retrieval
   - Accumulator information

#### Error Patterns
- **Common Error**: "ValueError: Header x-eieusercontext is required"
- **Error Status Codes**: status.code = 2 (ERROR)
- **HTTP Error Codes**: 400, 500 observed in downstream services
- **Exception Events**: Captured in span events with full stacktraces

#### Timing Patterns
- **Trace Durations**: Range from ~100ms to ~2000ms
- **Span Durations**: Vary significantly based on operation type
- **External API Calls**: Often the longest duration components

---

## 2. Data Model Design

### 2.1 Design Principles

1. **Star Schema**: Fact and dimension tables for analytical queries
2. **Normalization**: Separate tables for attributes and events to handle variability
3. **Partitioning**: Date-based partitioning for performance and cost optimization
4. **Clustering**: Strategic clustering on frequently queried columns
5. **Incremental Processing**: Support for incremental data loads
6. **Query Optimization**: Pre-aggregated tables for common analytical patterns

### 2.2 Data Model Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    STAGING LAYER                             │
│  trace_data_landing (Raw JSON as STRING)                    │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    FACT TABLES                               │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │ fact_spans   │  │fact_span_attrs   │  │fact_span_    │  │
│  │              │  │                  │  │events        │  │
│  └──────────────┘  └──────────────────┘  └──────────────┘  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                  DIMENSION TABLES                            │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │dim_services  │  │ dim_traces  │                         │
│  └──────────────┘  └──────────────┘                         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              AGGREGATED TABLES                               │
│  ┌────────────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │agg_service_perf    │  │agg_span_     │  │agg_trace_   │ │
│  │                    │  │dependencies │  │summary      │ │
│  └────────────────────┘  └──────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Core Fact Tables

#### 2.3.1 fact_spans

**Purpose**: Core span-level data with timing and status information

**Grain**: One row per span

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| span_id | STRING | NOT NULL | Unique span identifier (Primary Key component) |
| trace_id | STRING | NOT NULL | Trace identifier (groups related spans) |
| parent_span_id | STRING | NULL | Parent span ID (NULL for root spans) |
| span_name | STRING | NULL | Span operation name (e.g., "POST /costestimator/v1/rate") |
| span_kind | STRING | NULL | SPAN_KIND_SERVER, SPAN_KIND_CLIENT, SPAN_KIND_INTERNAL |
| service_name | STRING | NULL | Service that generated the span |
| start_time_unix_nano | INT64 | NULL | Start time in nanoseconds since epoch |
| end_time_unix_nano | INT64 | NULL | End time in nanoseconds since epoch |
| duration_ms | FLOAT64 | NULL | Calculated duration: (end - start) / 1,000,000 |
| status_code | INT64 | NULL | 0=OK, 1=ERROR, 2=UNSET |
| status_message | STRING | NULL | Error message if status_code != 0 |
| instrumentation_library_name | STRING | NULL | Library that instrumented the span |
| inserted_at | TIMESTAMP | NOT NULL | When the record was inserted (for partitioning) |

**Partitioning**: DATE(inserted_at)  
**Clustering**: trace_id, service_name

**Business Rules**:
- Root spans have parent_span_id = NULL or '0000000000000000'
- duration_ms calculated from start/end times
- status_code = 0 indicates success, != 0 indicates error

#### 2.3.2 fact_span_attributes

**Purpose**: Normalized key-value attributes for spans

**Grain**: One row per span attribute

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| span_id | STRING | NOT NULL | Foreign key to fact_spans |
| trace_id | STRING | NOT NULL | For filtering/joining efficiency |
| attribute_key | STRING | NOT NULL | Attribute name (e.g., "http.method", "request.membership_id") |
| attribute_value_string | STRING | NULL | String value if applicable |
| attribute_value_int | INT64 | NULL | Integer value if applicable |
| attribute_value_double | FLOAT64 | NULL | Double value if applicable |
| inserted_at | TIMESTAMP | NOT NULL | When the record was inserted |

**Partitioning**: DATE(inserted_at)  
**Clustering**: trace_id, span_id, attribute_key

**Business Rules**:
- Only one value type populated per row (string, int, or double)
- Common keys: http.*, request.*, response.*, function.*, error.*

#### 2.3.3 fact_span_events

**Purpose**: Events within spans (exceptions, logs, custom events)

**Grain**: One row per span event

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| span_id | STRING | NOT NULL | Foreign key to fact_spans |
| trace_id | STRING | NOT NULL | For filtering/joining efficiency |
| event_name | STRING | NULL | Event name (e.g., "exception") |
| event_time_unix_nano | INT64 | NULL | Event timestamp in nanoseconds |
| exception_type | STRING | NULL | Exception type if applicable (e.g., "ValueError") |
| exception_message | STRING | NULL | Exception message if applicable |
| exception_stacktrace | STRING | NULL | Full stacktrace if applicable |
| inserted_at | TIMESTAMP | NOT NULL | When the record was inserted |

**Partitioning**: DATE(inserted_at)  
**Clustering**: trace_id, span_id

**Business Rules**:
- Exception fields populated when event_name = 'exception'
- Stacktraces may contain newlines and special characters

### 2.4 Dimension Tables

#### 2.4.1 dim_services

**Purpose**: Service catalog with metadata

**Grain**: One row per unique service

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| service_name | STRING | NOT NULL | Primary key |
| service_version | STRING | NULL | Service version |
| deployment_environment | STRING | NULL | Environment (dev, prod, etc.) |
| cloud_provider | STRING | NULL | GCP, AWS, Azure, etc. |
| cloud_account_id | STRING | NULL | Cloud account identifier |
| cloud_platform | STRING | NULL | Platform (gcp_kubernetes_engine, etc.) |
| k8s_cluster_name | STRING | NULL | Kubernetes cluster name |
| first_seen_at | TIMESTAMP | NULL | First time this service appeared |
| last_seen_at | TIMESTAMP | NULL | Most recent appearance |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Clustering**: service_name

**Business Rules**:
- SCD Type 1: Updates overwrite previous values
- first_seen_at and last_seen_at track service lifecycle

#### 2.4.2 dim_traces

**Purpose**: Trace-level aggregations and metadata

**Grain**: One row per trace

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| trace_id | STRING | NOT NULL | Primary key |
| root_span_id | STRING | NULL | Root span (no parent) |
| service_name | STRING | NULL | Primary service for the trace |
| total_spans | INT64 | NULL | Total number of spans in trace |
| total_duration_ms | FLOAT64 | NULL | End-to-end trace duration |
| has_errors | BOOLEAN | NULL | Whether any span has error status |
| error_count | INT64 | NULL | Number of spans with errors |
| trace_start_time | TIMESTAMP | NULL | Trace start time |
| trace_end_time | TIMESTAMP | NULL | Trace end time |
| inserted_at | TIMESTAMP | NOT NULL | When the record was inserted |

**Partitioning**: DATE(inserted_at)  
**Clustering**: trace_id, service_name

**Business Rules**:
- total_duration_ms = MAX(end_time) - MIN(start_time) for all spans
- has_errors = TRUE if any span has status_code != 0

### 2.5 Aggregated Tables

#### 2.5.1 agg_service_performance

**Purpose**: Service-level performance metrics aggregated by time period

**Grain**: One row per service per hour per day

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| service_name | STRING | NOT NULL | Service identifier |
| metric_date | DATE | NOT NULL | Date of the metric |
| metric_hour | INT64 | NULL | Hour (0-23), NULL for daily aggregates |
| total_requests | INT64 | NULL | Total number of requests |
| avg_duration_ms | FLOAT64 | NULL | Average request duration |
| p50_duration_ms | FLOAT64 | NULL | 50th percentile duration |
| p95_duration_ms | FLOAT64 | NULL | 95th percentile duration |
| p99_duration_ms | FLOAT64 | NULL | 99th percentile duration |
| max_duration_ms | FLOAT64 | NULL | Maximum duration |
| min_duration_ms | FLOAT64 | NULL | Minimum duration |
| error_count | INT64 | NULL | Number of errors |
| error_rate | FLOAT64 | NULL | Error rate (0-1) |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Partitioning**: metric_date  
**Clustering**: service_name, metric_hour

**Business Rules**:
- Aggregated from dim_traces table
- Percentiles calculated using APPROX_QUANTILES
- error_rate = error_count / total_requests

#### 2.5.2 agg_span_dependencies

**Purpose**: Service-to-service dependency graph and call patterns

**Grain**: One row per source-to-target service call pattern per day

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| source_service | STRING | NOT NULL | Calling service |
| target_service | STRING | NOT NULL | Called service (extracted from http.url) |
| http_method | STRING | NULL | HTTP method if applicable |
| http_url_pattern | STRING | NULL | Normalized URL pattern |
| call_count | INT64 | NULL | Number of calls |
| avg_duration_ms | FLOAT64 | NULL | Average call duration |
| p95_duration_ms | FLOAT64 | NULL | 95th percentile duration |
| error_count | INT64 | NULL | Number of failed calls |
| error_rate | FLOAT64 | NULL | Error rate (0-1) |
| last_called_at | TIMESTAMP | NULL | Most recent call timestamp |
| metric_date | DATE | NOT NULL | Date of the metric |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Partitioning**: metric_date  
**Clustering**: source_service, target_service

**Business Rules**:
- target_service extracted from http.url attribute
- http_url_pattern normalized (removes query params, IDs)

#### 2.5.3 agg_trace_summary

**Purpose**: High-level trace summaries with business context

**Grain**: One row per trace

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| trace_id | STRING | NOT NULL | Trace identifier |
| service_name | STRING | NULL | Primary service |
| request_type | STRING | NULL | Extracted from http.method + http.route |
| membership_id | STRING | NULL | Business entity identifier |
| service_code | STRING | NULL | Medical service code (CPT4/HCPC) |
| benefit_product_type | STRING | NULL | Benefit product type (Medical, etc.) |
| total_duration_ms | FLOAT64 | NULL | End-to-end duration |
| span_count | INT64 | NULL | Number of spans |
| has_errors | BOOLEAN | NULL | Error flag |
| error_count | INT64 | NULL | Number of errors |
| trace_timestamp | TIMESTAMP | NULL | Trace start time |
| inserted_at | TIMESTAMP | NOT NULL | When the record was inserted |

**Partitioning**: DATE(trace_timestamp)  
**Clustering**: trace_id, service_name

**Business Rules**:
- Business context extracted from span attributes
- Enables business-user friendly queries

---

## 3. Data Transformation Strategy

### 3.1 Extraction Phase

**Objective**: Extract structured data from JSON strings

**Approach**:
- Use BigQuery JSON_EXTRACT functions (works with string JSON)
- Handle nested structures with JSON_EXTRACT_ARRAY
- UNNEST arrays to flatten hierarchical data

**Key Transformations**:
1. Extract spans from batches → fact_spans
2. Extract attributes from spans → fact_span_attributes
3. Extract events from spans → fact_span_events
4. Extract resource attributes → dim_services

### 3.2 Enrichment Phase

**Objective**: Calculate derived metrics and establish relationships

**Key Calculations**:
- Duration: `(end_time_unix_nano - start_time_unix_nano) / 1,000,000`
- Root span identification: `parent_span_id IS NULL OR parent_span_id = '0000000000000000'`
- Trace-level aggregations: COUNT, MIN, MAX, SUM
- Error detection: `status_code != 0`

### 3.3 Aggregation Phase

**Objective**: Pre-compute common analytical metrics

**Aggregations**:
- Service performance: Hourly/daily metrics with percentiles
- Dependencies: Service-to-service call patterns
- Trace summaries: Business-context enriched traces

### 3.4 Incremental Processing

**Strategy**: Process only new traces
- Use `inserted_at` timestamps
- Check for existing trace_id before insert
- Update dimension tables with latest values

---

## 4. Query Patterns and Use Cases

### 4.1 Performance Analysis

**Use Case**: Monitor service latency and throughput

**Example Queries**:
- Average latency by service over time
- P95/P99 latency trends
- Request volume by hour/day
- Slowest operations identification

**Tables Used**: `agg_service_performance`, `fact_spans`

### 4.2 Error Analysis

**Use Case**: Identify and analyze failures

**Example Queries**:
- Error rate by service
- Exception types and frequency
- Failed requests with business context
- Error correlation with latency

**Tables Used**: `fact_span_events`, `agg_trace_summary`, `fact_spans`

### 4.3 Dependency Analysis

**Use Case**: Understand service dependencies and call patterns

**Example Queries**:
- Service dependency graph
- Most called services
- Service call chains
- Bottleneck identification

**Tables Used**: `agg_span_dependencies`, `fact_span_attributes`

### 4.4 Business Context Analysis

**Use Case**: Correlate technical metrics with business entities

**Example Queries**:
- Performance by membership type
- Service code analysis
- Benefit product type patterns
- Request patterns by business entity

**Tables Used**: `agg_trace_summary`, `fact_span_attributes`

---

## 5. Implementation Recommendations

### 5.1 Table Creation

1. **Create all tables** using DDL scripts with proper partitioning and clustering
2. **Set up incremental processing** to avoid duplicate data
3. **Implement data quality checks** to validate transformations

### 5.2 Data Pipeline

1. **Extract**: Transform JSON to fact tables (parallel processing where possible)
2. **Enrich**: Populate dimension tables
3. **Aggregate**: Create aggregated tables for common queries
4. **Schedule**: Run hourly or as data arrives

### 5.3 Performance Optimization

1. **Partitioning**: All fact tables partitioned by date
2. **Clustering**: Strategic clustering on frequently queried columns
3. **Indexing**: Consider additional indexes for specific query patterns
4. **Materialized Views**: For very common query patterns

### 5.4 Data Retention

1. **Raw Data**: Retain for 90 days (configurable)
2. **Fact Tables**: Retain for 365 days
3. **Aggregated Tables**: Retain indefinitely (smaller size)
4. **Archive Strategy**: Move old data to cold storage

### 5.5 Monitoring and Maintenance

1. **Data Quality**: Monitor row counts, null rates, data freshness
2. **Performance**: Track query execution times
3. **Cost**: Monitor BigQuery slot usage and storage costs
4. **Alerts**: Set up alerts for pipeline failures

---

## 6. Data Quality Considerations

### 6.1 Data Completeness

- **Missing Attributes**: Some spans may not have all expected attributes
- **Truncated Data**: Response bodies may be truncated in source data
- **Null Handling**: Use COALESCE and NULLIF appropriately

### 6.2 Data Consistency

- **Trace Completeness**: Ensure all spans for a trace are loaded
- **Timing Consistency**: Validate start_time < end_time
- **Relationship Integrity**: Validate parent_span_id references

### 6.3 Data Validation Rules

1. **Span Validation**:
   - span_id and trace_id must not be NULL
   - duration_ms must be >= 0
   - status_code must be 0, 1, or 2

2. **Trace Validation**:
   - total_spans must be > 0
   - total_duration_ms must be >= 0
   - trace_start_time must be <= trace_end_time

3. **Attribute Validation**:
   - Only one value type (string/int/double) populated per row
   - attribute_key must not be NULL

---

## 7. Future Enhancements

### 7.1 Additional Dimensions

- **dim_providers**: Provider information from request attributes
- **dim_service_codes**: Medical service code catalog
- **dim_memberships**: Membership type and characteristics

### 7.2 Advanced Aggregations

- **agg_trace_flows**: Complete trace flow analysis
- **agg_error_patterns**: Error pattern recognition
- **agg_sla_compliance**: SLA compliance tracking

### 7.3 Real-time Analytics

- **Streaming Pipeline**: Process traces in near real-time
- **Real-time Dashboards**: Connect to BI tools for live monitoring
- **Alerting**: Automated alerts based on thresholds

---

## 8. Appendix

### 8.1 Sample Trace Statistics

Based on 5 sample traces analyzed:

- **Average Spans per Trace**: ~10-15 spans
- **Average Trace Duration**: 100-2000ms
- **Common Span Types**: SERVER (entry), CLIENT (external calls), INTERNAL (processing)
- **Error Rate**: ~80% of traces contain errors (sample bias)
- **Common Error**: "ValueError: Header x-eieusercontext is required"

### 8.2 Key Attributes Catalog

**HTTP Attributes**:
- http.method, http.url, http.status_code, http.route, http.path, http.scheme, http.host

**Request Attributes**:
- request.membership_id, request.service_code, request.benefit_product_type, request.body, request.num_providers

**Response Attributes**:
- response.body, response.type

**Function Attributes**:
- function.name, function.module, function.duration_ms

**Error Attributes**:
- error.type, error.message

### 8.3 External Services Identified

- `qaapi01.int.aetna.com/healthcare/qapath2/v5/privapp/auth/oauth2/token` (OAuth)
- `qaapi01.int.aetna.com/hcb/qapath2/hcb/v4/internal/servicesbenefits/details/retrieve` (Benefits)
- `qaapi01.int.aetna.com/hcb/qapath2/internal/v5/membershipsaccumulators` (Accumulators)

---

## Document Control

**Version History**:
- v1.0 (January 2025): Initial analysis and data model design

**Approvals**:
- [ ] Data Engineering Lead
- [ ] Architecture Review Board
- [ ] Business Stakeholders

**Next Steps**:
1. Review and approve data model
2. Create table DDL scripts
3. Develop transformation SQL scripts
4. Implement data pipeline (Airflow DAG)
5. Create analytical queries and dashboards

---

**End of Document**

