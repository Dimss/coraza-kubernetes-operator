# Coraza WAF Performance Testing

Comprehensive performance testing framework for evaluating Coraza Web Application Firewall overhead in Kubernetes/Istio environments.

## Overview

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

---

## Quick Start

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

---

## Architecture

### Test Flow

```
┌─────────────┐      ┌──────────────────┐      ┌─────────────┐      ┌─────────────┐
│   Fortio    │─────▶│ Istio Gateway    │─────▶│ Coraza WAF  │─────▶│   Backend   │
│ Load Gen    │      │ (Service Mesh)   │      │ (WASM Plugin)│      │ Echo Service│
└─────────────┘      └──────────────────┘      └─────────────┘      └─────────────┘
     500 QPS              TLS + Routing           Rule Evaluation       Response
```

---

## Data Sources

### Test Results Location

```
results/fixed-500qps+{profile}__waf-{config}__{timestamp}.json
```

**Examples:**
- `fixed-500qps+baseline-get__waf-off__20261012T093852Z.json`
- `fixed-500qps+args-heavy__waf-crs__20261012T101539Z.json`

### Profile Definitions

**Location**: `profiles/*.env`

- `baseline-get.env` - Simple GET request
- `args-heavy.env` - 25 query parameters  
- `headers-heavy.env` - 15 custom headers

### WAF Ruleset Definitions

**Location**: `rulesets/*.yaml`

- `minimal.yaml` - 2 basic rules
- `medium.yaml` - 30 custom rules
- `crs.yaml` - 563 OWASP CoreRuleSet rules
- `off` - No ruleset (baseline)

---

## Report Generation

### How It Works

```
Test Results (JSON) → generate-report.py → report-template.html → performance-report.html
```

### Generator Script

**File**: `generate-report.py`

**Main Functions:**
1. `parse_result()` - Extract metrics from Fortio JSON
2. `find_latest_result()` - Find most recent test for each combination
3. `prepare_chart_data()` - Generate Chart.js configurations
4. `generate_percentile_cards()` - Create P50/P95/P99 cards
5. `generate_test_matrix()` - Create test completion matrix

### Template

**File**: `templates/report-template.html`

**Sections:**
1. Architecture - Traffic flow diagram
2. Scenario - Test configuration
3. Profiles - Load profiles (with modal popups)
4. WAF Rulesets - Ruleset definitions (with modal popups)
5. Test Matrix - 3×4 grid of all combinations
6. Performance Results - Charts and visualizations
7. Findings - Key insights

---

## Visualizations

The report includes **7 interactive visualizations**:

### 1. Latency by Load Profile
**Type**: Bar chart  
**Shows**: p50 latency across WAF configs

### 2. CRS Overhead by Request Profile  
**Type**: Bar chart  
**Shows**: CRS overhead % vs baseline

### 3. Latency Distribution Across Percentiles
**Type**: Grouped bar chart  
**Shows**: p50, p75, p90, p95, p99 for all combinations

### 4. Request Complexity Impact ⭐
**Type**: Line chart  
**Shows**: How latency changes with request complexity  
**Insight**: CRS gets FASTER with complex requests!

### 5. Rules Efficiency Analysis
**Type**: Scatter plot  
**Shows**: Overhead per rule (economies of scale)

### 6. Tail Latency Gap
**Type**: Bar chart  
**Shows**: Predictability (p99 - p50 gap)  
**Colors**: Green (<10ms), Orange (<20ms), Red (≥20ms)

### 7. Latency Heatmap
**Type**: Color-coded table  
**Shows**: p95 latency across all 12 combinations  
**Colors**: Green (fast) → Red (slow)

---

## Data Structure

### Fortio JSON Result

```json
{
  "DurationHistogram": {
    "Count": 30000,
    "Percentiles": [
      {"Percentile": 50, "Value": 0.00280},
      {"Percentile": 75, "Value": 0.00410},
      {"Percentile": 90, "Value": 0.00497},
      {"Percentile": 95, "Value": 0.00819},
      {"Percentile": 99, "Value": 0.01631}
    ]
  },
  "ActualQPS": 500.0
}
```

### Parsed Metrics

```python
{
    'count': 30000,
    'p50': 2.80,
    'p75': 4.10,
    'p90': 4.97,
    'p95': 8.19,
    'p99': 16.31,
    'p99_9': 33.00,
    'avg': 3.47,
    'actual_qps': 500.0
}
```

### Results Dictionary

```python
results = {
    ('baseline-get', 'off'): {...},
    ('baseline-get', 'minimal'): {...},
    ...
    ('headers-heavy', 'crs'): {...}
}
# Total: 12 combinations
```

---

## Customization

### Add New Load Profile

1. Create `profiles/my-profile.env`
2. Update `profiles` list in `generate-report.py` (line ~615)
3. Regenerate report

### Add New WAF Config

1. Create `rulesets/my-ruleset.yaml`
2. Update `configs` list in `generate-report.py` (line ~616)
3. Regenerate report

### Add New Chart

1. Add canvas to `templates/report-template.html`
2. Generate config in `prepare_chart_data()` function
3. Initialize chart in JavaScript section

---

## Files Reference

| File | Purpose |
|------|---------|
| `generate-report.py` | Report generator (main script) |
| `templates/report-template.html` | HTML template |
| `results/performance-report.html` | Generated report |
| `analyze-results.py` | Data analysis tool |
| `DATA-STRUCTURE-VERIFICATION.md` | Data flow docs |

---

## Troubleshooting

### Missing Test Results

```bash
# Check result files
ls -1 results/fixed-500qps+*.json | wc -l
# Should show 12+

# Check for empty files
ls -lh results/fixed-500qps+*.json
```

### Chart Not Rendering

1. Open browser console (F12)
2. Check for JavaScript errors
3. Verify: `console.log(Object.keys(DATA))`

### Report Generation Fails

```bash
# Run with verbose output
python3 -u generate-report.py 2>&1 | tee report-gen.log

# Verify template exists
ls -l templates/report-template.html
```

---

## Dependencies

### Runtime
- Python 3.x
- Modern browser (Chrome/Firefox/Safari)

### Testing
- Kubernetes cluster
- Istio v1.27+
- Fortio v1.76+
- coraza-kubernetes-operator v0.5.0+

---

**Last Updated**: 2026-10-12  
**Generator**: generate-report.py  
**Template**: report-template.html
