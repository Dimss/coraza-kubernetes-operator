# Performance Testing Quick Reference

## Generate Report

```bash
python3 generate-report.py
```

Output: `results/performance-report.html`

---

## Data Sources

- **Test Results**: `results/fixed-500qps+{profile}__waf-{config}__{timestamp}.json`
- **Profiles**: `profiles/*.env`
- **Rulesets**: `rulesets/*.yaml`
- **Scenario**: `scenarios/fixed-500qps-short.env`

---

## Report Sections

1. **Architecture** - Traffic flow diagram
2. **Scenario** - Test configuration (QPS, duration, connections)
3. **Profiles** - Load profiles (baseline, args-heavy, headers-heavy)
4. **WAF Rulesets** - Security configs (off, minimal, medium, CRS)
5. **Test Matrix** - 3×4 grid (12 test combinations)
6. **Performance Results** - 7 interactive visualizations

---

## 7 Visualizations

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

## Key Metrics

| Metric | Description |
|--------|-------------|
| p50 | Median latency (50th percentile) |
| p75 | 75th percentile |
| p90 | 90th percentile |
| p95 | 95th percentile |
| p99 | 99th percentile |
| p99.9 | 99.9th percentile |

---

## Test Matrix

3 Profiles × 4 Configs = 12 Tests

```
             │ Off │ Minimal │ Medium │ CRS │
─────────────┼─────┼─────────┼────────┼─────┤
Baseline-GET │  ✓  │    ✓    │   ✓    │  ✓  │
Args-Heavy   │  ✓  │    ✓    │   ✓    │  ✓  │
Headers-Heavy│  ✓  │    ✓    │   ✓    │  ✓  │
```

---

## Files

| File | Purpose |
|------|---------|
| `generate-report.py` | Report generator |
| `templates/report-template.html` | HTML template |
| `results/performance-report.html` | Generated report |
| `README.md` | Full documentation |
| `DATA-STRUCTURE-VERIFICATION.md` | Technical details |

---

## Customization

### Add Profile
1. Create `profiles/new-profile.env`
2. Update `profiles` list in `generate-report.py`
3. Regenerate

### Add Ruleset
1. Create `rulesets/new-ruleset.yaml`
2. Update `configs` list in `generate-report.py`
3. Regenerate

---

## Troubleshooting

**Missing data?**
```bash
ls -1 results/fixed-500qps+*.json | wc -l  # Should show 12+
```

**Chart not rendering?**
- Open browser console (F12)
- Check: `console.log(Object.keys(DATA))`

**Generation fails?**
```bash
python3 -u generate-report.py 2>&1 | tee log.txt
```

---

## Dependencies

- Python 3.x
- Browser (Chrome/Firefox/Safari)
- Chart.js (loaded from CDN)

---

**Docs**: `README.md` | **Generated**: `results/performance-report.html`
