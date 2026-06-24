# SRE Troubleshooting Quick Reference - Steps & Durations

## Scenario 1: Cascading Product Catalog Failure
**Manual Investigation Duration: 30-60 minutes**

| Step | Name | Focus |
|------|------|-------|
| 1 | Detect the issue | Alerts: error rate > 5%, p99 latency > 5s |
| 2 | Check service health | Pod status, logs, service endpoints |
| 3 | Investigate root cause | Deployment issues, resource pressure, events |
| 4 | Look at OTEL traces | Jaeger: error rate, timeouts, slow queries |
| 5 | Check dependencies | Database health, connection pools, network |
| 6 | Correlate with metrics | Prometheus: latency spikes, error rates |
| 7 | Impact analysis | Which services affected, customer impact |
| 8 | Remediation | Restart service, scale up, circuit breaker |
| 9 | Verify recovery | Pod status, connectivity, error rate |
| 10 | Post-incident analysis | Timeline, root cause, prevention |

**Automated Scenario Duration: 5 minutes**

---

## Scenario 2: OOMKilled Cart Service
**Manual Investigation Duration: 1-2 hours**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify the issue | OOMKilled pods, CrashLoopBackOff, restarts |
| 2 | Investigate memory usage | Actual consumption vs requests/limits |
| 3 | Find memory leak | Application logs, allocation patterns |
| 4 | Analyze heap dump | Objects consuming memory, circular refs |
| 5 | Check request patterns | Rate spikes, large request correlation |
| 6 | Correlate with load | Traffic vs OOMKilled timing |
| 7 | Check dependencies | Slow services causing buffering |
| 8 | Immediate remediation | Increase limits, reduce concurrency, profile |
| 9 | Long-term fix | Fix memory leak in code |
| 10 | Verify recovery | Monitor memory, error rate, no more OOMKilled |

**Automated Scenario Duration: 10 minutes**

---

## Scenario 3: Image Pull Failure (Recommendation)
**Manual Investigation Duration: 20-45 minutes**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify the issue | ImagePullBackOff status |
| 2 | Check image registry | Does image exist in ECR/registry? |
| 3 | Check registry credentials | imagePullSecrets configured and valid |
| 4 | Check DNS resolution | Can pods resolve registry hostname? |
| 5 | Manual pull test | Try pulling image on node |
| 6 | Check build/push pipeline | Was image pushed? When? |
| 7 | Remediation options | Use different tag, re-push, fix credentials |
| 8 | Verify recovery | ImagePullBackOff → Running |
| 9 | Root cause analysis | Why wasn't image in registry? |
| 10 | Prevention | Image validation, tag pinning, alerts |

**Automated Scenario Duration: 5 minutes**

---

## Scenario 4: CPU Throttling (Ad Service)
**Manual Investigation Duration: 1-2 hours**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify throttling | CPU metrics: actual ≈ limits |
| 2 | Check CPU requests/limits | Are they appropriate for workload? |
| 3 | Analyze request patterns | Request rate spike correlation? |
| 4 | Check CPU-intensive ops | CPU profiling: hot paths, loops |
| 5 | Check resource contention | Other pods on same node? Node capacity? |
| 6 | Check for noisy neighbors | Any pods with excessive CPU? |
| 7 | Latency correlation | When latency spike vs CPU throttling? |
| 8 | Immediate remediation | Increase limits, scale replicas, affinity |
| 9 | Long-term optimization | Profile and optimize hot paths |
| 10 | Verify recovery | CPU usage, latency p99, no throttling |

**Automated Scenario Duration: 10 minutes**

---

## Scenario 5: Liveness Probe Failure (Frontend)
**Manual Investigation Duration: 30-60 minutes**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify the issue | CrashLoopBackOff, pod restarts |
| 2 | Check probe configuration | What is /health endpoint checking? |
| 3 | Check probe endpoint | Manually test: is it returning 200? |
| 4 | Check application logs | Errors before crash? Uncaught exceptions? |
| 5 | Check dependencies | Can reach database, cache, other services? |
| 6 | Check resource constraints | Pending pods? Node capacity issues? |
| 7 | Check recent changes | Recent deployment? What changed? |
| 8 | Immediate remediation | Rollback, adjust probe, disable (temp) |
| 9 | Fix underlying issue | Fix dependency connectivity/config |
| 10 | Verify recovery | Pod status: CrashLoopBackOff → Running |

**Automated Scenario Duration: 5 minutes**

---

## Scenario 6: Wrong Environment Variable (Checkout)
**Manual Investigation Duration: 20-45 minutes**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify checkout failures | Error rate spike, logs show config errors |
| 2 | Check environment variables | What env vars does pod have? |
| 3 | Check deployment definition | Are variables spelled correctly? Values right? |
| 4 | Check if from ConfigMap | Is ConfigMap value correct? |
| 5 | Verify connectivity | Can checkout reach payment service? |
| 6 | Check startup logs | Full logs from pod creation, config shown? |
| 7 | Check recent deployments | When did wrong config get deployed? |
| 8 | Immediate remediation | Fix var, update ConfigMap, restart pods |
| 9 | Verify fix | Manually test payment endpoint |
| 10 | Prevent future issues | Config validation, typed config, tests |

**Automated Scenario Duration: 5 minutes**

---

## Scenario 7: Database Connection Failure (Ad Service)
**Manual Investigation Duration: 45-90 minutes**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify DB connectivity issue | Logs: "connection refused", timeout |
| 2 | Check database pod status | Is database pod running? |
| 3 | Test database connectivity | Can ad pod connect to postgres? |
| 4 | Check DB resource capacity | CPU/memory high? At limits? |
| 5 | Check connection pool | Active connections vs max allowed |
| 6 | Check query performance | Any slow queries? Locks? |
| 7 | Check network connectivity | Pod networking OK? Firewall? DNS? |
| 8 | Check connection string | Hostname, port, credentials correct? |
| 9 | Immediate remediation | Scale DB, increase limits, kill idle conns |
| 10 | Verify recovery | Connection test succeeds, error rate → 0% |

**Automated Scenario Duration: 10 minutes**

---

## Scenario 8: Noisy Neighbor (Recommendation)
**Manual Investigation Duration: 30-60 minutes**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify resource contention | Top pods by CPU/memory |
| 2 | Identify affected services | Which services degraded? Correlate timing |
| 3 | Check node allocation | Noisy neighbor and victim on same node? |
| 4 | Check pod resource limits | Does noisy neighbor have limits? Or unlimited? |
| 5 | Identify root cause | Legitimate spike or runaway process? |
| 6 | Check for memory leaks | Long-running process growing memory? |
| 7 | Immediate remediation | Add limits, isolate to dedicated node, kill |
| 8 | Verify recovery | CPU/memory normal, affected services recover |
| 9 | Long-term mitigation | Set limits, QoS classes, pod priorities |
| 10 | Monitoring | Alert on high resource pods, contention |

**Automated Scenario Duration: 10 minutes**

---

## Scenario 9: Autoscaler Node Pressure
**Manual Investigation Duration: N/A (Capacity Planning)**
**Automated Scenario Duration: 10 minutes (Validation)**

| Step | Name | Focus |
|------|------|-------|
| 1 | Identify resource pressure | Pending pods, insufficient CPU/memory |
| 2 | Check cluster capacity | Total available vs used resources |
| 3 | Check autoscaler status | Is autoscaler running? Check logs |
| 4 | Monitor node creation | Are new nodes being created? Check ASG |
| 5 | Check node readiness | Are new nodes joining cluster? Ready state? |
| 6 | Monitor pod scheduling | As nodes come online, pods schedule |
| 7 | Verify autoscaler config | scale-down-enabled? delays? |
| 8 | Check performance impact | Latency during scaling? Error rate? |
| 9 | Monitor scale-down | After peak, nodes removed? Timeline? |
| 10 | Post-scaling analysis | How long? Any requests dropped? |

---

## Scenario 10: Combined App + Infra Failure
**Manual Investigation Duration: 2-4 hours**

| Step | Name | Focus |
|------|------|-------|
| 1 | Multi-layer diagnosis | App failures + infra failures together |
| 2 | Timeline correlation | What failed first? How did it cascade? |
| 3 | Application layer diagnosis | Slow productcatalog? DB connectivity? |
| 4 | Infrastructure layer diagnosis | Memory pressure? Pod evictions? |
| 5 | Causal analysis | Which layer caused the other? Both? |
| 6 | Identify blast radius | Which services affected? How many users? |
| 7 | OTEL trace analysis | Jaeger: latency spike, retries, timeouts |
| 8 | Metric correlation | Latency + memory + evictions timeline |
| 9 | Remediation strategy | Fix app layer first, then infra layer |
| 10 | Post-incident analysis | Root cause, prevention, blast radius |

**Automated Scenario Duration: 15 minutes**

---

## Summary Table: All Scenarios

| # | Scenario | Steps | Manual Time | Automated Time | Complexity |
|---|----------|-------|-------------|----------------|------------|
| 1 | Cascading failure | 10 | 30-60m | 5m | Medium |
| 2 | OOMKilled | 10 | 1-2h | 10m | High |
| 3 | Image pull failure | 10 | 20-45m | 5m | Medium |
| 4 | CPU throttling | 10 | 1-2h | 10m | High |
| 5 | Liveness probe | 10 | 30-60m | 5m | Medium |
| 6 | Wrong env var | 10 | 20-45m | 5m | Low |
| 7 | DB connection | 10 | 45-90m | 10m | High |
| 8 | Noisy neighbor | 10 | 30-60m | 10m | High |
| 9 | Autoscaler | 10 | N/A | 10m | Medium |
| 10 | Combined failure | 10 | 2-4h | 15m | Very High |

---

## Time Savings Analysis

**Total Manual Investigation Time (if all scenarios happened in production)**
- Best case (all quick): 30-60 + 20-45 + 20-45 + 30-60 + 20-45 + 45-90 + 30-60 + N/A + 2-4h = **7-11 hours**
- Likely case (medium scenarios): **12-20 hours**
- Worst case (all happen together): **25+ hours**

**Total Automated Scenario Time**
- All 10 scenarios: 5 + 10 + 5 + 10 + 5 + 5 + 10 + 10 + 10 + 15 = **85 minutes (~1.5 hours)**

**Training Benefit**
- Learn all scenarios: 1.5 hours
- Per incident prevented: saves 2-4 hours in production
- ROI: 1.5h investment prevents 20+ hours of firefighting

---

## Step Categories (Across All Scenarios)

### Common Investigation Steps
1. **Identify the issue** - Detect symptoms (appears in all 10)
2. **Check service/resource status** - Look at pods, resources (appears in 9/10)
3. **Check logs** - Application logs (appears in 8/10)
4. **Check dependencies** - Database, external services (appears in 7/10)
5. **Check correlation** - Metrics, events, timing (appears in 8/10)
6. **Immediate remediation** - Quick fix (appears in 10/10)
7. **Verify recovery** - Confirm fix worked (appears in 10/10)
8. **Root cause analysis** - Why did it happen? (appears in 8/10)
9. **Prevention** - How to avoid next time (appears in 10/10)
10. **Monitoring/Alerts** - Catch early (appears in 9/10)

### Skills Developed
- ✅ Kubernetes troubleshooting (all scenarios)
- ✅ Log analysis (8/10 scenarios)
- ✅ Metrics analysis (9/10 scenarios)
- ✅ Distributed tracing (6/10 scenarios)
- ✅ Resource management (8/10 scenarios)
- ✅ Database debugging (3/10 scenarios)
- ✅ Network troubleshooting (4/10 scenarios)
- ✅ Performance analysis (4/10 scenarios)
- ✅ Multi-layer diagnosis (1/10 scenarios - the hardest!)

---

## Recommended Learning Path

**Week 1: Foundation**
- Scenario 6 (Wrong env var) - 20m
- Scenario 3 (Image pull) - 20m
- Scenario 5 (Liveness probe) - 30m

**Week 2: Resource Issues**
- Scenario 2 (OOMKilled) - 30m
- Scenario 4 (CPU throttling) - 30m
- Scenario 8 (Noisy neighbor) - 30m

**Week 3: Connectivity & State**
- Scenario 7 (DB connection) - 30m
- Scenario 1 (Cascading failure) - 30m
- Scenario 9 (Autoscaler) - 30m

**Week 4: Advanced**
- Scenario 10 (Combined failure) - 45m

**Total Time Investment: 5.5 hours**
**Skills Gained: Emergency troubleshooting for all major failure modes**

---

## Execution Guide

### For Individual SRE Practice
```bash
Monday: Run Scenario 6 (20 min) → Review steps
Tuesday: Run Scenario 3 (20 min) → Review steps
Wednesday: Run Scenario 5 (30 min) → Deep dive
Thursday: Run Scenario 2 (30 min) → Memory debugging
Friday: Run Scenario 4 (30 min) → Performance analysis
```

### For Team Training
```
Session 1: Scenarios 3, 5, 6 (Low complexity - 1 hour)
Session 2: Scenarios 2, 4, 8 (Medium complexity - 1.5 hours)
Session 3: Scenarios 1, 7, 9 (High complexity - 2 hours)
Session 4: Scenario 10 (Very high - 1 hour with discussion)
```

### For Onboarding New SREs
- Day 1: Scenario 6 (config issues)
- Day 2: Scenario 3 (deployment issues)
- Day 3: Scenario 5 (health check issues)
- Day 4: Scenario 1 (service troubleshooting)
- Day 5: Scenario 10 (comprehensive assessment)
