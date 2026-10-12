# Coraza WAF Performance Testing

Comprehensive performance testing framework for the Coraza Kubernetes Operator.

## Quick Start

```bash
# Generate report from existing results
python3 generate-report.py

# View report
open results/performance-report.html
```

## Documentation

📚 **Complete Guide**: See **[GUIDE.md](GUIDE.md)** for comprehensive documentation including:

- Overview & Quick Start
- Architecture
- Terminology & Concepts
- Data Structure & Verification
- Test Scenarios
- Metrics & Observability
- Report Generation
- Customization
- Troubleshooting
- References

## Test Matrix

**3 Load Profiles × 4 Rulesets = 12 Tests**

- **Profiles**: baseline-get, args-heavy, headers-heavy
- **Rulesets**: off, minimal (2 rules), medium (30 rules), CRS (563 rules)
- **Scenario**: Fixed 500 QPS for 60 seconds

## Key Files

| File | Purpose |
|------|---------|
| `GUIDE.md` | Complete documentation (all sections) |
| `generate-report.py` | Report generator |
| `templates/report-template.html` | HTML template |
| `results/performance-report.html` | Generated report |
| `manifests/` | K8s deployment manifests |
| `profiles/` | Load profile definitions |
| `scenarios/` | Load scenario configs |

## Test Environment

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

**For complete documentation, see [GUIDE.md](GUIDE.md)**
