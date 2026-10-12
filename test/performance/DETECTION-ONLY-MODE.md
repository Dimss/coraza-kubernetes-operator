# Detection-Only Mode Configuration

**Date**: 2026-10-12  
**Change**: All WAF RuleSets configured for detection-only mode  
**Reason**: Accurate performance overhead measurement

---

## What Changed

### Before
```yaml
SecRuleEngine On
```
- WAF evaluated rules and **blocked** matching requests (403 Forbidden)
- Early-exit on first blocking rule
- Measured: "Time to find blocking rule" (varies by ruleset)

### After
```yaml
SecRuleEngine DetectionOnly
```
- WAF evaluates **ALL** rules (no early-exit)
- Rule matches are **logged only** (not blocked)
- All traffic passes through (200 OK)
- Measured: "Full rule evaluation overhead"

---

## Files Modified

| File | Line Changed | Status |
|------|--------------|--------|
| `manifests/rulesets/minimal.yaml` | Line 9 | ✅ Updated |
| `manifests/rulesets/medium.yaml` | Line 11 | ✅ Updated |
| `manifests/rulesets/crs.yaml` | Line 24 | ✅ Updated |

---

## Why This Matters

### Previous Test Results (Blocking Mode)

```
Profile         │ Off    │ Minimal │ Medium  │ CRS    │
────────────────┼────────┼─────────┼─────────┼────────┤
baseline-get    │ 39% ❌ │ 100% ❌ │ 100% ❌ │ 0% ✓   │
args-heavy      │ 51% ❌ │ 100% ❌ │ 100% ❌ │ 100% ❌│
headers-heavy   │ 0% ✓   │ 0% ✓    │ 0% ✓    │ 32% ❌ │

Latency (baseline-get):
  Minimal (2 rules):  4.09ms ← Blocked at rule #1 or #2
  Medium (30 rules):  3.83ms ← Blocked at rule #8 (never evaluated #9-30)
```

**Problem**: Medium appeared "faster" because it blocked earlier, not because it was more efficient.

### Expected Results (Detection-Only Mode)

```
Profile         │ Off    │ Minimal │ Medium  │ CRS    │
────────────────┼────────┼─────────┼─────────┼────────┤
ALL profiles    │ 0%  ✓  │ 0%  ✓   │ 0%  ✓   │ 0%  ✓  │

Latency (baseline-get):
  Minimal (2 rules):   X.XXms ← Evaluated all 2 rules
  Medium (30 rules):   Y.YYms ← Evaluated all 30 rules
  CRS (563 rules):     Z.ZZms ← Evaluated all 563 rules
  
Expected: Medium > Minimal, CRS > Medium (more rules = more overhead)
```

---

## Next Steps

### 1. Redeploy RuleSets

```bash
# Apply updated RuleSets to cluster
kubectl apply -f manifests/rulesets/minimal.yaml
kubectl apply -f manifests/rulesets/medium.yaml
kubectl apply -f manifests/rulesets/crs.yaml

# Verify deployment
kubectl get rulesets -n waf-perf
kubectl get rulesources -n waf-perf
```

### 2. Wait for Propagation

```bash
# WAF plugin needs to reload configuration
# Wait ~30 seconds for pods to sync
kubectl get pods -n waf-perf -w
```

### 3. Re-run Performance Tests

```bash
# Run all test combinations
./run-all-tests.sh

# Or run specific profile/config
./run-test.sh baseline-get minimal
./run-test.sh baseline-get medium
./run-test.sh baseline-get crs
```

### 4. Verify Results

```bash
# Check latest test results
ls -lt results/fixed-500qps+*.json | head -5

# Verify 0% error rate
python3 << 'PYEOF'
import json
import glob

files = glob.glob('results/fixed-500qps+baseline-get__waf-minimal__*.json')
latest = sorted(files, reverse=True)[0]

with open(latest) as f:
    data = json.load(f)

codes = data.get('RetCodes', {})
total = sum(codes.values())
errors = sum(v for k, v in codes.items() if k != '200')

print(f"File: {latest}")
print(f"Return codes: {codes}")
print(f"Error rate: {errors}/{total} ({errors/total*100:.1f}%)")
print("")
if errors == 0:
    print("✅ SUCCESS: 0% error rate (detection-only working)")
else:
    print(f"❌ FAILURE: {errors/total*100:.1f}% error rate")
PYEOF
```

### 5. Generate New Report

```bash
# Generate updated HTML report
python3 generate-report.py

# View results
open results/performance-report.html
```

---

## Expected Outcomes

### Latency Trends

With detection-only mode, you should see:

1. **Consistent 0% error rate** across all tests
2. **Linear overhead scaling** with rule count:
   - Off (0 rules): Baseline
   - Minimal (2 rules): Small overhead
   - Medium (30 rules): ~15× minimal overhead
   - CRS (563 rules): ~18× medium overhead

3. **Request complexity behavior**:
   - Simple requests (baseline-get): Higher per-rule cost (empty collections)
   - Complex requests (args-heavy): Lower per-rule cost (rules process data efficiently)
   - CRS may still show inverse relationship (gets faster with complex requests)

### Data Quality Indicators

✅ **Valid test**:
- Error rate: 0%
- All requests: 200 OK
- Actual QPS ≈ Requested QPS (±5%)

❌ **Invalid test**:
- Error rate > 0%
- Mixed 200/403 responses
- Should re-run

---

## Troubleshooting

### RuleSet Not Updating

```bash
# Force ruleset refresh
kubectl delete ruleset perf-ruleset -n waf-perf
kubectl apply -f manifests/rulesets/minimal.yaml

# Restart affected pods
kubectl rollout restart deployment loadgen -n waf-perf
```

### Still Seeing 403 Errors

1. **Check RuleSource status**:
   ```bash
   kubectl get rulesources -n waf-perf -o yaml | grep -A 5 "SecRuleEngine"
   ```

2. **Check Engine configuration**:
   ```bash
   kubectl get engines -n waf-perf -o yaml
   ```

3. **Check WasmPlugin**:
   ```bash
   kubectl get wasmplugins -n istio-system -o yaml
   ```

### Verify Detection-Only Active

```bash
# Check operator logs
kubectl logs -n coraza-operator-system deployment/coraza-operator-controller-manager -f

# Should see: "SecRuleEngine DetectionOnly" in ruleset reconciliation
```

---

## Reverting to Blocking Mode

If you need to test blocking behavior:

```bash
# Change back to blocking
sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/g' manifests/rulesets/*.yaml

# Reapply
kubectl apply -f manifests/rulesets/
```

**Note**: Blocking mode results are NOT comparable across rulesets due to early-exit behavior.

---

## References

- [Coraza SecRuleEngine Directive](https://coraza.io/docs/seclang/directives/#secruleengine)
- [OWASP CRS Detection-Only Mode](https://coreruleset.org/docs/deployment/install/#detection-only-mode)

---

**Status**: ✅ All RuleSets configured for detection-only mode  
**Next**: Redeploy RuleSets → Re-run tests → Verify 0% error rate → Generate new report
