# Performance Testing Terminology

## Two Independent Dimensions

Your performance tests vary **TWO independent variables**:

### 1️⃣ Load Profile (Request Shape)

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

**In Code:**
```bash
# baseline-get.env
METHOD=""
REQ_PATH="/"
PAYLOAD_SIZE="0"

# args-heavy.env
METHOD=""
REQ_PATH="/search?q=shoes&page=2&sort=price&..."  # 25 params

# headers-heavy.env
METHOD=""
REQ_PATH="/"
HEADERS=("X-Request-ID: abc123" "X-Session-ID: xyz789" ...)  # 15 headers
```

---

### 2️⃣ WAF Configuration (Ruleset)

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

**In Code:**
```yaml
# minimal.yaml
apiVersion: waf.k8s.coraza.io/v1alpha1
kind: RuleSet
spec:
  sources:
  - name: perf-base          # Engine initialization
  - name: perf-block-attack  # Blocks "attack" keyword

# crs.yaml
apiVersion: waf.k8s.coraza.io/v1alpha1
kind: RuleSet
spec:
  sources:
  - name: perf-crs  # All 563 OWASP CoreRuleSet rules
```

---

## Complete Test Matrix

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

---

## Why Both Dimensions Matter

### The Key Insight

**CRS overhead depends on REQUEST SHAPE:**

| Test Combination | p50 Latency | Overhead | Why |
|------------------|-------------|----------|-----|
| **baseline-get + crs** | 2.80ms → 8.55ms | **+205%** | CRS rules iterate **empty** ARGS (wasted work) |
| **args-heavy + crs** | 3.48ms → 4.55ms | **+31%** | CRS rules inspect 25 params (useful work) |
| **headers-heavy + crs** | 2.03ms → 2.31ms | **+14%** | Rules inspect headers efficiently |

### Why This Happens

OWASP CoreRuleSet has ~165 rules like this:

```
SecRule ARGS "@rx attack" "id:1,phase:2,deny"
```

**With baseline-get (no query params):**
- ARGS collection is **empty**
- Rule still executes 165 times iterating nothing
- Pure overhead ❌

**With args-heavy (25 query params):**
- ARGS collection has **25 items**
- Rule executes 165 times × 25 params = real inspection
- Overhead is amortized across useful work ✅

---

## Terminology Summary

| Term | What It Controls | Example | File Location |
|------|------------------|---------|---------------|
| **Load Profile** | Request shape | `baseline-get`, `args-heavy`, `headers-heavy` | `manifests/load-profiles/*.env` |
| **Ruleset** | WAF rules to evaluate | `off`, `minimal`, `medium`, `crs` | `manifests/rulesets/*.yaml` |
| **Scenario** | Load pattern (QPS, duration, connections) | `fixed-500qps` (500 QPS, 30s, 16 conns) | `scenarios/*.env` |

---

## In Reports

**Current dashboards:** One per load profile, showing all rulesets
- `dashboard-baseline-get.html` → baseline-get profile, all 4 rulesets
- `dashboard-args-heavy.html` → args-heavy profile, all 4 rulesets
- `dashboard-headers-heavy.html` → headers-heavy profile, all 4 rulesets

**Why separate dashboards?**
- Template designed to show: **one profile, multiple rulesets**
- Can't easily show: **multiple profiles, multiple rulesets** on one chart
- Each dashboard answers: "For THIS request shape, how do different WAF configs perform?"

---

## Improving Report Clarity

To avoid confusion, we should:

1. **Consistently use:**
   - "Load Profile" (not "profile" alone)
   - "Ruleset" or "WAF Configuration" (not "config" alone)

2. **In dashboards:**
   - Title: "Load Profile: Baseline-Get" (make it prominent)
   - Subtitle: "Comparing WAF Rulesets" (clarify what's being compared)

3. **In comprehensive report:**
   - Show both dimensions clearly
   - Use headers like "### Load Profile: baseline-get" not just "### baseline-get"

---

*This terminology guide should be referenced when creating or reading performance reports.*
