# Performance Test Execution Checklist

**Goal**: Run comprehensive performance tests and generate HTML report

---

## Prerequisites

### 0. Install Istio (if not already installed)

**For OpenShift clusters:**

```bash
# Use the helper script which detects platform and installs Sail Operator
../../hack/install-istio.sh
```

**For Kubernetes clusters:**

```bash
# Use the helper script which installs upstream Istio
../../hack/install-istio.sh
```

Alternatively, see [DEVELOPMENT.md](../../DEVELOPMENT.md) for manual installation steps.

### 1. Cluster Access

```bash
# Verify cluster connection
kubectl cluster-info

# Verify Istio is installed
kubectl get pods -n istio-system

# Check namespace
kubectl get ns waf-perf 2>/dev/null || kubectl create ns waf-perf
```

### 2. Deploy RuleSets (Detection-Only Mode)

```bash
# Deploy all rulesets
kubectl apply -f manifests/rulesets/minimal.yaml
kubectl apply -f manifests/rulesets/medium.yaml
kubectl apply -f manifests/rulesets/crs.yaml

# Verify deployment
kubectl get rulesets -n waf-perf
kubectl get rulesources -n waf-perf
```

### 3. Verify Detection-Only Mode

```bash
# Check all rulesets have DetectionOnly
kubectl get rulesources -n waf-perf -o yaml | grep -A 2 "SecRuleEngine"

# Should show: SecRuleEngine DetectionOnly
```

### 4. Deploy Test Infrastructure

```bash
# Deploy backend, gateway, loadgen
kubectl apply -f manifests/backend.yaml
kubectl apply -f manifests/gateway.yaml
kubectl apply -f manifests/loadgen.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=backend -n waf-perf --timeout=120s
kubectl wait --for=condition=ready pod -l app=loadgen -n waf-perf --timeout=120s
```

### 5. Deploy Engines

```bash
# Deploy engine (links RuleSet to Gateway)
kubectl apply -f manifests/engine.yaml

# Verify WasmPlugin created
kubectl get wasmplugins -n istio-system
```

### 6. Wait for Propagation

```bash
# Wait ~30 seconds for:
# - RuleSet reconciliation
# - WasmPlugin deployment
# - Envoy config reload
sleep 30
```

---

## Run Tests

### Execute Comprehensive Test Suite

```bash
# Run all 12 test combinations (3 profiles × 4 rulesets)
./run-comprehensive-tests.sh
```

**Expected duration**: ~20-30 minutes

**What it does**:
1. Runs baseline-get profile (off, minimal, medium, crs)
2. Runs args-heavy profile (off, minimal, medium, crs)
3. Runs headers-heavy profile (off, minimal, medium, crs)
4. Validates results (0% error rate expected)
5. Generates comprehensive HTML report

---

## Validate Results

### Check Test Results

```bash
# Count result files
ls -1 results/fixed-500qps+*__waf-*__*.json | wc -l
# Should show: 12 or more

# Check for errors
python3 << 'PYEOF'
import json
import glob

files = glob.glob('results/fixed-500qps+*__waf-*__*.json')
print(f"Total files: {len(files)}\n")

errors_found = False
for f in sorted(files)[-12:]:  # Check latest 12
    try:
        with open(f) as fh:
            data = json.load(fh)
        codes = data.get('RetCodes', {})
        total = sum(codes.values())
        errors = sum(v for k, v in codes.items() if k != '200')
        
        if errors > 0:
            print(f"❌ {f}: {errors}/{total} errors ({errors/total*100:.1f}%)")
            errors_found = True
        else:
            print(f"✅ {f}: 0 errors")
    except Exception as e:
        print(f"⚠️  {f}: Parse error - {e}")

if not errors_found:
    print("\n✅ All tests have 0% error rate!")
PYEOF
```

### View HTML Report

```bash
# Generate report (if not auto-generated)
python3 generate-report.py

# Open report
open results/performance-report.html

# Check report sections:
# - Architecture
# - Scenario
# - Profiles
# - WAF Rulesets
# - Test Matrix (should show 12 green checkmarks)
# - Performance Results (7 visualizations)
```

---

## Expected Results

### Test Matrix Status

```
                │ Off │ Minimal │ Medium │ CRS │
────────────────┼─────┼─────────┼────────┼─────┤
Baseline-GET    │  ✓  │    ✓    │   ✓    │  ✓  │
Args-Heavy      │  ✓  │    ✓    │   ✓    │  ✓  │
Headers-Heavy   │  ✓  │    ✓    │   ✓    │  ✓  │
```

### Error Rates

All tests should show **0% error rate** (all 200 OK responses).

If you see 403 errors:
- ❌ Detection-only mode not working
- ❌ Check RuleSet configuration
- ❌ Verify `SecRuleEngine DetectionOnly` in all rulesets

### Latency Patterns

**Expected with detection-only mode**:
- Off < Minimal < Medium < CRS (linear scaling)
- All configs achieve ~500 QPS (no throughput degradation)
- CRS may show inverse relationship with request complexity

---

## Troubleshooting

### No Cluster Access

```bash
# Check kubeconfig
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>
```

### Pods Not Ready

```bash
# Check pod status
kubectl get pods -n waf-perf

# Check pod logs
kubectl logs -n waf-perf deployment/loadgen
kubectl logs -n waf-perf deployment/backend
```

### High Error Rates

```bash
# Verify detection-only mode
grep "SecRuleEngine" manifests/rulesets/*.yaml

# Should show: SecRuleEngine DetectionOnly (NOT "On")

# Reapply if needed
kubectl apply -f manifests/rulesets/

# Wait for propagation
sleep 30

# Re-run tests
./run-comprehensive-tests.sh
```

### Script Fails

```bash
# Check lib.sh is present
ls -l lib.sh

# Check scripts directory
ls -l scripts/

# Run with debug
bash -x ./run-comprehensive-tests.sh
```

---

## Manual Test Execution

If the comprehensive script fails, run tests manually:

```bash
# Test one profile/ruleset combination
LOAD_PROFILE=baseline-get ./scripts/suite.sh scenarios/fixed-500qps.env minimal medium crs

# Test another profile
LOAD_PROFILE=args-heavy ./scripts/suite.sh scenarios/fixed-500qps.env minimal medium crs

# Test third profile
LOAD_PROFILE=headers-heavy ./scripts/suite.sh scenarios/fixed-500qps.env minimal medium crs

# Generate report
python3 generate-report.py
```

---

## Success Criteria

✅ All 12 tests completed  
✅ 0% error rate across all tests  
✅ Results JSON files created  
✅ HTML report generated  
✅ Report shows all visualizations  
✅ Test matrix shows 12 green checkmarks  

---

**Once all criteria are met, the goal is complete!**
