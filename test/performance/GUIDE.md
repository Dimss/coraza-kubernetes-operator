# Coraza WAF Performance Testing - Complete Guide

**Project**: Coraza Kubernetes Operator  
**Component**: Performance Testing Framework  
**Last Updated**: 2026-10-12  
**Version**: 1.0

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Quick Start](#2-quick-start)
- [3. Architecture](#3-architecture)
- [4. Terminology & Concepts](#4-terminology--concepts)
- [5. Data Structure & Verification](#5-data-structure--verification)
- [6. Test Scenarios](#6-test-scenarios)
- [7. Metrics & Observability](#7-metrics--observability)
- [8. Report Generation](#8-report-generation)
- [9. Customization](#9-customization)
- [10. Troubleshooting](#10-troubleshooting)
- [11. References](#11-references)

---

## 1. Overview

### Purpose

This framework tests WAF performance across multiple dimensions:
- **3 Load Profiles**: Different HTTP request shapes (baseline-get, args-heavy, headers-heavy)
- **4 WAF Configurations**: Security vs performance tradeoffs (off, minimal, medium, CRS)
- **1 Load Scenario**: Fixed 500 QPS for 60 seconds
- **= 12 Test Combinations**: Complete performance matrix

### Key Findings

The testing revealed **counterintuitive behavior**:
- CRS (563 rules) gets **FASTER** with more complex requests
- Simple GET: 8.55ms → Complex GET: 2.31ms (73% faster!)
- Reason: Rules work efficiently on real data vs iterating empty collections
- Throughput impact: **ZERO** (all configs achieve 500 QPS)

### Test Environment

```
Kubernetes Cluster (OpenShift / K8s)
├── Istio + Gateway API
├── Coraza Operator (control plane)
├── RuleSetCache (dynamic rule delivery)
├── Gateway pods (Envoy + coraza-proxy-wasm)
├── Backend services
├── Load generator (Fortio)
└── Prometheus + Grafana
```

---

## 2. Quick Start

### Generate Report from Existing Results

```bash
# Generate HTML report
python3 generate-report.py

# Open report
open results/performance-report.html
```

### Run New Performance Tests

```bash
# 1. Run all test combinations (takes ~8 hours)
./run-all-tests.sh

# 2. Generate report
python3 generate-report.py

# 3. View results
open results/performance-report.html
```

### File Locations

- **Test Results**: `results/fixed-500qps+{profile}__waf-{config}__{timestamp}.json`
- **Profiles**: `profiles/*.env`
- **Rulesets**: `manifests/rulesets/*.yaml`
- **Scenario**: `scenarios/fixed-500qps-short.env`
- **Generator**: `generate-report.py`
- **Template**: `templates/report-template.html`

---

## 3. Architecture

### 3.1 Product Architecture

CKO is a **two-plane** system:

```
+----------------------------------+         +------------------------------------+
|         CONTROL PLANE            |         |          DATA PLANE                |
|  (Operator manager pod)          |         |  (Envoy + WASM in Gateway pods)    |
|                                  |         |                                    |
|  - RuleSource/RuleSet/Engine     |         |  - Intercepts every HTTP request   |
|    reconcilers                   |         |  - Runs Coraza WAF evaluation      |
|  - Compiles + validates SecLang  |         |  - Blocks / allows / detects       |
|  - RuleSetCache HTTP server      |←─poll───|  - Polls cache for rule updates    |
|    (:18080, in-memory, versioned)|──rules─→|  - Emits coraza_waf_* metrics      |
|  - Applies WasmPlugin via SSA    |         |                                    |
|  Metrics: controller-runtime +   |         |  Metrics: coraza_waf_* on Envoy    |
|  coraza_* on :8443               |         |  stats port :15090                 |
+----------------------------------+         +------------------------------------+
```

**Key Architectural Fact:** The Coraza engine runs as a **WebAssembly (WASM) module**
inside the Envoy proxy of each Gateway pod. Every HTTP request is evaluated inline
by this WASM filter. This is the most performance-critical component.

### 3.2 Test Flow

```
┌─────────────┐      ┌──────────────────┐      ┌─────────────┐      ┌─────────────┐
│   Fortio    │─────▶│ Istio Gateway    │─────▶│ Coraza WAF  │─────▶│   Backend   │
│ Load Gen    │      │ (Service Mesh)   │      │ (WASM Plugin)│      │ Echo Service│
└─────────────┘      └──────────────────┘      └─────────────┘      └─────────────┘
     500 QPS              TLS + Routing           Rule Evaluation       Response
```

### 3.3 Control Flow (Rules)

```
kubectl apply: RuleSource / RuleData / RuleSet / Engine (CRs)
      │
      ▼
RuleSourceReconciler ──► validate SecLang ──► status: Validated/InvalidRules
      │
      ▼
RuleSetReconciler ──► aggregate + validate ──► RuleSetCache (in-memory)
      │
      ▼
EngineReconciler ──► WasmPlugin (SSA) ──► Istio injects WASM into Gateway pods
      │
      ▼
Gateway pod polls RuleSetCache every 15s ──► loads rules into WASM filter
      │
      ▼
HTTP request arrives ──► WASM filter evaluates rules ──► allow/block/detect
```

### 3.4 Components Under Test

#### Tier 1: Request Hot Path (Latency Critical)

| Component | Where | Measurement |
|-----------|-------|-------------|
| **Coraza WASM filter** | Gateway pod (Envoy) | Added latency per request |
| **Rule evaluation** | WASM filter | p50/p95/p99 latency by ruleset size |
| **Istio overhead** | Gateway pod | Baseline (WAF off) measurement |

#### Tier 2: Rule Delivery Path

| Component | Where | Measurement |
|-----------|-------|-------------|
| **RuleSetCache** | Operator pod (:18080) | Cache hit rate, memory usage |
| **Poll loop** | Gateway WASM | Poll frequency vs freshness |

#### Tier 3: Control-Plane Scalability

| Component | Where | Measurement |
|-----------|-------|-------------|
| **RuleSource reconciler** | Operator | Validation time per rule count |
| **RuleSet reconciler** | Operator | Compilation time, memory usage |
| **Engine reconciler** | Operator | WasmPlugin apply time |

#### Tier 4: Rule-Update Propagation

| Component | Where | Measurement |
|-----------|-------|-------------|
| **End-to-end freshness** | CR update → Gateway enforcement | Total propagation latency |

### 3.5 Test Environment

```
                         ┌──────────────────────────────────────────────┐
                         │           Kubernetes cluster (SUT)           │
                         │                                              │
  ┌────────────┐  load   │  ┌────────────┐   WAF-filtered   ┌────────┐  │
  │  Load gen  │────────►│  │ Gateway pod│─────────────────►│ Backend│  │
  │ (fortio)   │  HTTP   │  │  Envoy +   │                  │ (echo) │  │
  └────────────┘         │  │ coraza-wasm│◄──poll rules──┐  └────────┘  │
        │                │  └────────────┘               │              │
        │                │                          ┌────┴───────────┐  │
        │                │                          │ Operator pod   │  │
        │                │  reconcile Engine/RuleSet│ (control plane)│  │
        │                │  + RuleSetCache :18080   │  cache server  │  │
        │                │                          └────────────────┘  │
        │                │                                              │
        │                │  ┌───────────────────────────────────────┐   │
        │                │  │ Prometheus + Grafana                  │   │
        │                │  │  - ServiceMonitor → operator :8443    │   │
        │                │  │  - PodMonitor → Envoy stats :15090    │   │
        │                │  └───────────────────────────────────────┘   │
        │                └──────────────────────────────────────────────┘
        │
   results/JSON ──► generate-report.py ──► performance-report.html
```

**Environment Requirements:**

- Kubernetes **v1.32+** (or OpenShift **v4.20+**)
- Istio + Gateway API CRDs
- **Dedicated node pools**: Load generator, Gateway pods, and backend on separate nodes
- **Fixed resource requests/limits**: Prevent CPU throttling
- **Load generator in-cluster**: Minimize network latency
- Prometheus retention: ≥1 hour at ≤15s scrape interval

---

## 4. Terminology & Concepts

### 4.1 Two Independent Dimensions

Your performance tests vary **TWO independent variables**:

#### Load Profile (Request Shape)

**Definition:** The shape/characteristics of the HTTP request being sent

**Controls:**
- URL path and query parameters
- HTTP headers
- Request body
- HTTP method

**Location:** `manifests/load-profiles/*.env`

**Examples:**

| Profile | What It Sends | Why |
|---------|---------------|-----|
| **baseline-get** | `GET /` (no args, no extra headers) | WAF best case - minimal data to inspect |
| **args-heavy** | `GET /search?q=shoes&page=2&...` (25 params) | Exercises CRS ARGS rules (~165 rules iterate) |
| **headers-heavy** | `GET /` with 15 extra headers | Exercises REQUEST_HEADERS rules |

#### WAF Configuration (Ruleset)

**Definition:** Which WAF rules are being evaluated by Coraza

**Controls:**
- Number of rules
- Rule complexity
- What attack patterns to detect

**Location:** `manifests/rulesets/*.yaml`

**Examples:**

| Ruleset | Rules | Description |
|---------|-------|-------------|
| **off** | 0 | No WAF (baseline - Istio/Envoy only) |
| **minimal** | 2 | Engine on + basic attack blocking |
| **medium** | 30 | Typical hand-written security policy |
| **crs** | 563 | Full OWASP CoreRuleSet 4.7.0 |

### 4.2 Complete Test Matrix

**3 Load Profiles × 4 Rulesets = 12 Tests**

```
                    ┌─────────┬─────────┬────────┬─────┐
                    │   off   │ minimal │ medium │ crs │
┌───────────────────┼─────────┼─────────┼────────┼─────┤
│ baseline-get      │ Test 1  │ Test 2  │ Test 3 │ T 4 │
│ (simple GET)      │         │         │        │     │
├───────────────────┼─────────┼─────────┼────────┼─────┤
│ args-heavy        │ Test 5  │ Test 6  │ Test 7 │ T 8 │
│ (25 query params) │         │         │        │     │
├───────────────────┼─────────┼─────────┼────────┼─────┤
│ headers-heavy     │ Test 9  │ Test 10 │ Test 11│ T12 │
│ (15 extra headers)│         │         │        │     │
└───────────────────┴─────────┴─────────┴────────┴─────┘
```

### 4.3 Why Both Dimensions Matter

**CRS overhead depends on REQUEST SHAPE:**

| Test Combination | p50 Latency | Overhead | Why |
|------------------|-------------|----------|-----|
| **baseline-get + crs** | 2.80ms → 8.55ms | **+205%** | CRS rules iterate **empty** ARGS (wasted work) |
| **args-heavy + crs** | 3.48ms → 4.55ms | **+31%** | CRS rules inspect 25 params (useful work) |
| **headers-heavy + crs** | 2.03ms → 2.31ms | **+14%** | Rules inspect headers efficiently |

**Why This Happens:**

OWASP CoreRuleSet has ~165 rules like:
```
SecRule ARGS "@rx attack" "id:1,phase:2,deny"
```

- **With baseline-get (no query params)**: ARGS collection is empty → rule executes 165 times iterating nothing → pure overhead ❌
- **With args-heavy (25 query params)**: ARGS collection has 25 items → rule executes 165 times × 25 params → real inspection ✅

### 4.4 Detection-Only Mode

**Configuration**: `SecRuleEngine DetectionOnly` (vs `SecRuleEngine On`)

**Purpose**: Measure WAF evaluation overhead without blocking behavior

**Behavior:**
- ✅ All rules are evaluated (no early-exit)
- ✅ Rule matches are logged
- ✅ All traffic passes through (200 OK)
- ✅ True overhead measurement
- ✅ Apples-to-apples comparison

**Why It Matters:**

In blocking mode (`SecRuleEngine On`):
- Minimal (2 rules): 4.09ms ← Blocked at rule #1-2, stopped
- Medium (30 rules): 3.83ms ← Blocked at rule ~#8, never evaluated #9-30
- **Medium appears "faster"** but it's misleading!

In detection-only mode:
- Minimal (2 rules): X.XXms ← Evaluated all 2 rules
- Medium (30 rules): Y.YYms ← Evaluated all 30 rules
- **Medium > Minimal** (correct!)

---

## 5. Data Structure & Verification

### 5.1 Data Flow

```
Fortio JSON Results
    ↓
parse_result() in generate-report.py
    ↓
results dict: {(profile, config): {...metrics...}}
    ↓
prepare_chart_data(results, profiles, configs)
    ↓
DATA object with 7 chart configurations
    ↓
Template: report-template.html
    ↓
Generated HTML: performance-report.html
```

### 5.2 Test Result Format (Fortio JSON)

**Location**: `results/fixed-500qps+{profile}__waf-{config}__{timestamp}.json`

**Structure**:
```json
{
  "DurationHistogram": {
    "Count": 30000,
    "Avg": 0.00347,
    "Percentiles": [
      {"Percentile": 50, "Value": 0.00280},
      {"Percentile": 75, "Value": 0.00410},
      {"Percentile": 90, "Value": 0.00497},
      {"Percentile": 95, "Value": 0.00819},
      {"Percentile": 99, "Value": 0.01631},
      {"Percentile": 99.9, "Value": 0.03300}
    ]
  },
  "ActualQPS": 500.0,
  "RetCodes": {"200": 30000}
}
```

### 5.3 Parsed Metrics

**Function**: `parse_result(filepath)`

**Returns**:
```python
{
    'count': 30000,           # Total requests
    'p50': 2.80,              # ms
    'p75': 4.10,              # ms
    'p90': 4.97,              # ms
    'p95': 8.19,              # ms
    'p99': 16.31,             # ms
    'p99_9': 33.00,           # ms
    'avg': 3.47,              # ms
    'actual_qps': 500.0,      # Requests per second
    'codes': {'200': 30000}   # HTTP status codes
}
```

### 5.4 Results Dictionary

**Structure**:
```python
results = {
    ('baseline-get', 'off'): {p50: 2.80, p95: 8.19, p99: 16.31, ...},
    ('baseline-get', 'minimal'): {...},
    ...
    ('headers-heavy', 'crs'): {...}
}
```

**Total combinations**: 3 profiles × 4 configs = 12 entries

### 5.5 Chart Data Object

**Structure in HTML**:
```javascript
DATA = {
    // Original charts
    latencyChart: {...},
    overheadChart: {...},
    
    // Advanced visualizations
    latencyDistributionChart: {...},
    complexityScalingChart: {...},
    rulesEfficiencyChart: {...},
    tailGapChart: {...},
    heatmapData: [...],
    
    // Modal content
    rulesets: {
        off: "...",
        minimal: "...",
        medium: "...",
        crs: "..."
    }
}
```

### 5.6 Data Quality Indicators

✅ **Valid test**:
- Error rate: 0%
- All requests: 200 OK
- Actual QPS ≈ Requested QPS (±5%)

❌ **Invalid test**:
- Error rate > 0%
- Mixed 200/403 responses
- Should re-run

---

## 6. Test Scenarios

### 6.1 Load Patterns

| Scenario | QPS | Duration | Connections | Purpose |
|----------|-----|----------|-------------|---------|
| **fixed-500qps** | 500 | 60s | 16 | Standard overhead measurement |
| **ramp** | 0→1000 | 300s | Variable | Find saturation point |
| **soak** | 350 (70% max) | 1-4h | 16 | Detect memory leaks, GC issues |

### 6.2 Workload Model

**Traffic Profiles:**
| Profile | Description | Purpose |
|---------|-------------|---------|
| **Benign / passing** | Requests that match no rule (200) | Baseline WAF overhead on legitimate traffic |
| **Malicious / blocking** | Requests that trigger deny (403) | Cost of blocking path |
| **Detect-only** | Requests scored but not blocked | Full-evaluation cost |

**Payload Dimensions:**
- **Body size**: 0 B, 1 KB, 10 KB, 100 KB, 1 MB
- **Header count**: Low vs high
- **Method mix**: GET (no body) vs POST/PUT (body inspection)

### 6.3 Ruleset Dimensions

| Ruleset | Size | Represents |
|---------|------|------------|
| **Minimal** | ~2 rules | Floor / overhead of WASM filter itself |
| **Medium** | ~30 rules | Typical hand-written policy |
| **OWASP CRS** | ~563 rules | Realistic production WAF |

---

## 7. Metrics & Observability

### 7.1 Data-Plane (WAF) Metrics

**Source**: Envoy stats port `:15090`  
**Prefix**: `coraza_waf_*`  
**Scrape**: PodMonitor (optional)

**Key Metrics:**
- `coraza_waf_transactions_total` - Total requests processed
- `coraza_waf_interruptions_total` - Blocked requests
- `coraza_waf_transaction_duration_seconds` - Rule evaluation time

### 7.2 Control-Plane Metrics

**Source**: Operator pod `:8443`  
**Prefix**: `coraza_*` + controller-runtime  
**Scrape**: ServiceMonitor (optional)

**Key Metrics:**
- `controller_runtime_reconcile_total` - Reconciliation count
- `controller_runtime_reconcile_time_seconds` - Reconciliation duration
- `coraza_ruleset_cache_size_bytes` - Cache memory usage

### 7.3 Load-Generator Metrics

**Source**: Fortio JSON output  
**File**: `results/fixed-500qps+{profile}__waf-{config}__{timestamp}.json`

**Key Metrics:**
- Latency percentiles (p50, p75, p90, p95, p99, p99.9)
- Actual QPS achieved
- HTTP status code distribution
- Request count

### 7.4 Resource Metrics

**CPU Usage:**
- Gateway pods (Envoy + WASM)
- Operator pod
- Backend pods

**Memory Usage:**
- WASM heap size
- RuleSetCache size
- Operator memory

---

## 8. Report Generation

### 8.1 How It Works

```
Test Results (JSON) → generate-report.py → report-template.html → performance-report.html
```

### 8.2 Generator Script

**File**: `generate-report.py`

**Main Functions:**
1. `parse_result()` - Extract metrics from Fortio JSON
2. `find_latest_result()` - Find most recent test for each combination
3. `prepare_chart_data()` - Generate Chart.js configurations
4. `generate_percentile_cards()` - Create P50/P95/P99 cards
5. `generate_test_matrix()` - Create test completion matrix

### 8.3 Report Sections

1. **Architecture** - Traffic flow diagram
2. **Scenario** - Test configuration (QPS, duration, connections)
3. **Profiles** - Load profiles (with modal popups)
4. **WAF Rulesets** - Ruleset definitions (with modal popups)
5. **Test Matrix** - 3×4 grid of all combinations
6. **Performance Results** - Charts and visualizations
7. **Findings** - Key insights

### 8.4 Visualizations (7 Interactive Charts)

| # | Name | Type | Shows |
|---|------|------|-------|
| 1 | Latency by Load Profile | Bar | p50 across configs |
| 2 | CRS Overhead | Bar | CRS overhead % |
| 3 | Latency Distribution | Grouped bar | p50-p99 all combos |
| 4 | Complexity Impact ⭐ | Line | How CRS scales |
| 5 | Rules Efficiency | Scatter | ms per rule |
| 6 | Tail Latency Gap | Bar | p99-p50 gap |
| 7 | Heatmap | Table | p95 visual grid |

---

## 9. Customization

### 9.1 Add New Load Profile

1. Create profile file:
   ```bash
   cp profiles/baseline-get.env profiles/my-profile.env
   # Edit my-profile.env with custom parameters
   ```

2. Update generator:
   ```python
   # In generate-report.py, line ~615
   profiles = ['baseline-get', 'args-heavy', 'headers-heavy', 'my-profile']
   ```

3. Regenerate report

### 9.2 Add New WAF Config

1. Create ruleset:
   ```bash
   cp manifests/rulesets/minimal.yaml manifests/rulesets/my-ruleset.yaml
   # Edit my-ruleset.yaml
   ```

2. Update generator:
   ```python
   # In generate-report.py, line ~616
   configs = ['off', 'minimal', 'medium', 'crs', 'my-ruleset']
   ```

3. Regenerate report

### 9.3 Change Load Scenario

Edit `scenarios/fixed-500qps-short.env`:
```bash
QPS=1000              # Increase to 1000 QPS
DURATION=120s         # Longer test
CONNECTIONS=32        # More connections
```

Update file pattern in `find_latest_result()` if scenario name changes.

### 9.4 Add New Chart

1. Add canvas to template:
   ```html
   <canvas id="myChart"></canvas>
   ```

2. Generate config in `prepare_chart_data()`:
   ```python
   my_chart = {
       'type': 'bar',
       'data': {...},
       'options': {...}
   }
   ```

3. Add to return dict:
   ```python
   return {
       ...,
       'myChart': my_chart
   }
   ```

4. Initialize in template:
   ```javascript
   if (DATA.myChart) {
       const ctx = document.getElementById('myChart').getContext('2d');
       new Chart(ctx, DATA.myChart);
   }
   ```

---

## 10. Troubleshooting

### 10.1 Missing Test Results

**Symptom**: Report shows "N/A" or missing data

**Solutions**:
```bash
# Check result files exist
ls -1 results/fixed-500qps+*.json | wc -l
# Should be 12+

# Check file sizes (empty files are skipped)
ls -lh results/fixed-500qps+*.json

# Verify JSON is valid
python3 -m json.tool results/fixed-500qps+baseline-get__waf-off__*.json | head
```

### 10.2 Chart Not Rendering

**Symptom**: Blank chart area in report

**Solutions**:
1. Open browser console (F12) for JavaScript errors
2. Verify DATA object has chart configuration:
   ```javascript
   console.log(Object.keys(DATA));
   ```
3. Verify canvas element exists:
   ```javascript
   console.log(document.getElementById('latencyChart'));
   ```

### 10.3 High Error Rates

**Symptom**: Tests show high % of 403/500 errors

**Causes**:
- WAF in blocking mode (not detection-only)
- Rules triggering on test traffic
- Backend overloaded

**Solutions**:
1. Verify detection-only mode:
   ```bash
   grep "SecRuleEngine" manifests/rulesets/*.yaml
   # Should show: SecRuleEngine DetectionOnly
   ```

2. Check RuleSet deployment:
   ```bash
   kubectl get rulesets -n waf-perf -o yaml
   ```

3. Check backend logs:
   ```bash
   kubectl logs -n waf-perf deployment/backend
   ```

### 10.4 Report Generation Fails

**Symptom**: `generate-report.py` crashes

**Solutions**:
```bash
# Run with verbose output
python3 -u generate-report.py 2>&1 | tee report-gen.log

# Check Python version
python3 --version  # Should be 3.7+

# Verify template exists
ls -l templates/report-template.html
```

### 10.5 Inconsistent Results

**Symptom**: Same test shows different latencies across runs

**Causes**:
- CPU throttling
- Network variance
- Other workloads on nodes
- Insufficient warm-up

**Solutions**:
1. Pin Gateway pods to dedicated nodes
2. Set CPU requests = limits (guaranteed QoS)
3. Add warm-up period before measurement
4. Run multiple iterations and take median

---

## 11. References

### 11.1 File Structure

```
test/performance/
├── manifests/                  # K8s deployment manifests
│   ├── backend.yaml
│   ├── engine.yaml
│   ├── gateway.yaml
│   ├── loadgen.yaml
│   ├── load-profiles/
│   └── rulesets/
├── profiles/                   # Load profile definitions
├── scenarios/                  # Load scenario configs
├── scripts/                    # Test execution scripts
├── templates/                  # Report template
│   └── report-template.html
├── results/                    # Test results (JSON)
├── generate-report.py          # Report generator
├── analyze-results.py          # Data analysis
└── GUIDE.md                    # This file
```

### 11.2 Dependencies

**Runtime:**
- Python 3.7+
- Modern browser (Chrome/Firefox/Safari)

**Testing:**
- Kubernetes v1.32+ or OpenShift v4.20+
- Istio v1.27+
- Fortio v1.76+
- coraza-kubernetes-operator v0.5.0+

**Optional:**
- Prometheus (metrics collection)
- Grafana (live dashboards)

### 11.3 External Links

- [Fortio Documentation](https://github.com/fortio/fortio)
- [Chart.js Documentation](https://www.chartjs.org/docs/)
- [Coraza WAF](https://coraza.io/)
- [OWASP CoreRuleSet](https://coreruleset.org/)
- [Istio Performance](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/)
- [Coraza SecRuleEngine](https://coraza.io/docs/seclang/directives/#secruleengine)

### 11.4 Key Files Reference

| File | Purpose | Size |
|------|---------|------|
| `generate-report.py` | Report generator script | ~34KB |
| `templates/report-template.html` | HTML template | ~37KB |
| `results/performance-report.html` | Generated report | ~1MB |
| `analyze-results.py` | Data analysis tool | ~9KB |
| `GUIDE.md` | This comprehensive guide | ~40KB |

---

**Last Updated**: 2026-10-12  
**Version**: 1.0  
**Generator**: generate-report.py  
**Template**: report-template.html

