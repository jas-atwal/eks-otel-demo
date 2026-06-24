# SRE Troubleshooting Guide: Manual Investigation Steps for Each Scenario

This document shows what a good SRE would need to do **manually** to troubleshoot each failure scenario. It demonstrates the value of automated chaos testing - these steps that take hours in production can be practiced in minutes.

---

## Scenario 1: Cascading Product Catalog Failure

**What happens:** Product catalog service fails/times out → Frontend can't load product data → Cart service blocked → Full service cascade

### Manual SRE Investigation Steps

#### Step 1: Detect the issue (what alerts fire)
```
⚠️ Alert: Frontend error rate > 5%
⚠️ Alert: Product catalog p99 latency > 5s
⚠️ Alert: Frontend request queue backing up
```

#### Step 2: Check service health
```bash
# SSH to production bastion
ssh prod-bastion

# Check pod status
kubectl get pods -n otel-demo -l app=productcatalog

# Check logs for errors
kubectl logs -n otel-demo -l app=productcatalog --tail=50
# Look for: connection timeouts, database errors, panic

# Check service endpoints
kubectl get endpoints -n otel-demo productcatalog
# Verify pods are actually registered
```

#### Step 3: Investigate the root cause
```bash
# Check if it's a deployment issue
kubectl describe pod <productcatalog-pod> -n otel-demo
# Look for: ImagePullBackOff, CrashLoopBackOff, Pending status

# Check resource pressure
kubectl top pods -n otel-demo -l app=productcatalog
# High CPU/memory?

# Check recent events
kubectl get events -n otel-demo --sort-by='.lastTimestamp' | grep productcatalog
```

#### Step 4: Look at OTEL traces
```bash
# Open Jaeger UI
# Service: productcatalog
# Look for:
#   - Increased error rate
#   - Timeout exceptions
#   - Slow database queries
#   - Connection pool exhaustion
```

#### Step 5: Check dependencies
```bash
# Is the database healthy?
kubectl get pods -n otel-demo -l app=postgres
kubectl logs -n otel-demo -l app=postgres --tail=20

# Are there connection limits?
kubectl exec -it <postgres-pod> -- psql -c "SELECT count(*) FROM pg_stat_activity;"

# Check network connectivity
kubectl exec -it <productcatalog-pod> -- nc -zv postgres 5432
```

#### Step 6: Correlate with metrics
```bash
# In Prometheus, query:
histogram_quantile(0.99, http_request_duration_seconds{service="productcatalog"})
# Should see spike at incident time

rate(grpc_server_handled_total{grpc_service="ProductCatalogService"}[5m])
# Check error rate
```

#### Step 7: Impact analysis
```bash
# Which services are affected?
# Frontend calls productcatalog
# Cart calls productcatalog
# Checkout calls productcatalog

# Check frontend error cascade
kubectl logs -n otel-demo -l app=frontend | grep -i "productcatalog\|error" | tail -20

# How many customers affected?
# Check request patterns before/after incident
```

#### Step 8: Remediation
```bash
# Option 1: Restart the service
kubectl rollout restart deployment/productcatalog -n otel-demo

# Option 2: Scale up to handle load
kubectl scale deployment productcatalog --replicas=5 -n otel-demo

# Option 3: Add circuit breaker (if available)
kubectl patch deployment productcatalog -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"productcatalog","env":[{"name":"CIRCUIT_BREAKER_ENABLED","value":"true"}]}]}}}}'

# Option 4: Drain connections to old pods
kubectl delete pod <productcatalog-pod> -n otel-demo
```

#### Step 9: Verify recovery
```bash
# Check pod status
kubectl get pods -n otel-demo -l app=productcatalog

# Verify connectivity
kubectl exec -it <productcatalog-pod> -- curl http://localhost:8080/health

# Check error rate returning to normal
# Monitor: error_rate should drop to near 0%
# Monitor: p99 latency should return to baseline
```

#### Step 10: Post-incident analysis
```bash
# Generate timeline
kubectl get events -n otel-demo --sort-by='.lastTimestamp' | grep -E "productcatalog|frontend"

# Export traces for analysis
# (depends on your Jaeger setup)

# Questions to answer:
# - Why did productcatalog fail?
# - Why didn't circuit breaker catch it?
# - How long until auto-recovery?
# - What could have prevented this?
```

### Prevention Strategies
- ✅ Add circuit breaker to frontend→catalog calls
- ✅ Increase productcatalog replicas (redundancy)
- ✅ Add health checks with fast-fail
- ✅ Set connection pool limits
- ✅ Add timeout configurations
- ✅ Implement bulkhead pattern

### Time to Resolution
**Manual investigation:** 30-60 minutes
**With automated scenario:** 5 minutes (practice run)

---

## Scenario 2: OOMKilled Cart Service

**What happens:** Cart service memory limit exceeded → Pod killed → Requests fail → Service unavailable → Recovery via restart

### Manual SRE Investigation Steps

#### Step 1: Identify the issue
```bash
# Check for OOMKilled pods
kubectl get pods -n otel-demo -l app=cart -o wide

# Look for status: OOMKilled, CrashLoopBackOff
# Restart count will be high
```

#### Step 2: Investigate memory usage
```bash
# Check actual memory consumption
kubectl top pods -n otel-demo -l app=cart

# Check memory requests vs limits
kubectl get pods -n otel-demo -l app=cart -o yaml | grep -A 10 "resources:"

# Look for: requests < actual usage < limits
```

#### Step 3: Find memory leak
```bash
# Check application logs for memory growth patterns
kubectl logs -n otel-demo -l app=cart --previous --tail=100
# Look for: memory allocation patterns, leak indicators

# Check memory allocation timeline
# Was it gradual (leak) or sudden (spike)?
kubectl describe pod <cart-pod> -n otel-demo | grep -A 20 "Events:"
```

#### Step 4: Analyze heap dump (if available)
```bash
# Generate heap dump
kubectl exec -it <cart-pod> -- jcmd <pid> GC.heap_dump /tmp/heap.bin

# Extract and analyze
kubectl cp otel-demo/<cart-pod>:/tmp/heap.bin ./heap.bin

# Use tool to analyze:
# - What objects are consuming memory?
# - Any circular references?
# - Unclosed resources?
```

#### Step 5: Check for abnormal request patterns
```bash
# Query Prometheus for request rate spike
rate(http_requests_total{service="cart"}[5m])
# Did request rate increase before OOMKilled?

# Check request size distribution
histogram_quantile(0.99, http_request_size_bytes{service="cart"})
# Any unusually large requests?
```

#### Step 6: Correlate with load
```bash
# Check if OOMKilled correlates with high traffic
# Query: requests per second during incident

# Check if it's due to specific operation
# Analyze logs: which endpoints were called?

# Check if concurrent requests exceeded expectations
# Look at request queue depth
```

#### Step 7: Check dependencies
```bash
# Is cart buffering data from slow services?
# Check database query performance
kubectl logs -n otel-demo -l app=postgres | grep "slow query"

# Check cache hit rate
# (if using Redis or similar)
```

#### Step 8: Immediate remediation
```bash
# Option 1: Increase memory limit
kubectl set resources deployment cart \
  -n otel-demo \
  --limits=memory=512Mi

# Option 2: Reduce concurrent connections
kubectl patch deployment cart -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"cart","env":[{"name":"MAX_CONCURRENT_REQUESTS","value":"50"}]}]}}}}'

# Option 3: Enable memory profiling for next cycle
kubectl patch deployment cart -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"cart","env":[{"name":"MEMORY_PROFILING_ENABLED","value":"true"}]}]}}}}'
```

#### Step 9: Long-term fix
```bash
# Fix the memory leak in code
# - Review recent changes to cart service
# - Check for unclosed connections/resources
# - Add explicit garbage collection triggers
# - Profile on staging before deploying

# Deploy fixed version
kubectl rollout restart deployment/cart -n otel-demo
```

#### Step 10: Verify recovery
```bash
# Monitor memory usage
kubectl top pods -n otel-demo -l app=cart --watch

# Check error rate
# Should return to near 0%

# Verify no more OOMKilled events
kubectl get events -n otel-demo | grep -i oom
```

### Prevention Strategies
- ✅ Set appropriate memory requests/limits based on load testing
- ✅ Enable memory profiling in staging
- ✅ Monitor memory trends over time
- ✅ Set up alerts for memory usage > 80%
- ✅ Implement request throttling
- ✅ Use connection pooling with limits

### Time to Resolution
**Manual investigation:** 1-2 hours (especially finding leak)
**With automated scenario:** 10 minutes

---

## Scenario 3: Image Pull Failure (Recommendation Service)

**What happens:** Container image not found → ImagePullBackOff → Pod stuck pending → Service unavailable

### Manual SRE Investigation Steps

#### Step 1: Identify the issue
```bash
# Check pod status
kubectl get pods -n otel-demo -l app=recommendation

# Status: ImagePullBackOff
# Event: "Failed to pull image: image not found"
```

#### Step 2: Check image registry
```bash
# What image is the pod trying to pull?
kubectl describe pod <recommendation-pod> -n otel-demo | grep -A 5 "Image:"
# Example: 123456789.dkr.ecr.eu-west-2.amazonaws.com/recommendation:v1.2.3

# Does the image exist?
aws ecr describe-images \
  --repository-name recommendation \
  --region eu-west-2 \
  --image-ids imageTag=v1.2.3
```

#### Step 3: Check registry credentials
```bash
# Are the imagePullSecrets configured?
kubectl get pods -n otel-demo -l app=recommendation -o yaml | grep -A 5 "imagePullSecrets:"

# Are the secrets valid?
kubectl get secrets -n otel-demo -o yaml | grep -A 10 ecr-registry

# Can pods authenticate to ECR?
kubectl create serviceaccount test -n otel-demo
kubectl run -it test --image=busybox --serviceaccount=test -n otel-demo -- sh
# Inside: wget <image-registry-url> (should work or auth fail, not connection refused)
```

#### Step 4: Check DNS resolution
```bash
# Can pods resolve the registry hostname?
kubectl run -it debug --image=busybox -n otel-demo -- nslookup 123456789.dkr.ecr.eu-west-2.amazonaws.com

# Can they reach the registry?
kubectl run -it debug --image=busybox -n otel-demo -- nc -zv 123456789.dkr.ecr.eu-west-2.amazonaws.com 443
```

#### Step 5: Manual pull test
```bash
# Try pulling the image manually on a node
ssh ec2-user@<node-ip>
docker pull 123456789.dkr.ecr.eu-west-2.amazonaws.com/recommendation:v1.2.3

# If fails, check docker credentials
cat ~/.docker/config.json

# Try ECR login
aws ecr get-login-password --region eu-west-2 | docker login --username AWS --password-stdin 123456789.dkr.ecr.eu-west-2.amazonaws.com
docker pull 123456789.dkr.ecr.eu-west-2.amazonaws.com/recommendation:v1.2.3
```

#### Step 6: Check image build/push pipeline
```bash
# Did the image get pushed to the registry?
aws ecr describe-images --repository-name recommendation --region eu-west-2

# Check if the tag exists
aws ecr list-images --repository-name recommendation --region eu-west-2

# When was it pushed?
aws ecr describe-image-detail --repository-name recommendation --region eu-west-2
```

#### Step 7: Remediation options
```bash
# Option 1: Use a different image tag that exists
kubectl set image deployment/recommendation \
  recommendation=123456789.dkr.ecr.eu-west-2.amazonaws.com/recommendation:v1.2.2 \
  -n otel-demo

# Option 2: Re-push the image
# (trigger CI/CD rebuild)
# Then update deployment to use new tag

# Option 3: Update imagePullSecrets if credentials are the issue
kubectl patch serviceaccount default -n otel-demo \
  -p '{"imagePullSecrets": [{"name": "ecr-registry"}]}'
```

#### Step 8: Verify recovery
```bash
# Check pod status
kubectl get pods -n otel-demo -l app=recommendation

# Should transition: ImagePullBackOff → Pulling → Running

# Verify service is responding
kubectl exec -it <any-pod> -n otel-demo -- curl http://recommendation:8080/health
```

#### Step 9: Root cause analysis
```bash
# Why did the image not get pushed?
# - Build job failed?
# - Push job failed?
# - Wrong registry?
# - Wrong tag?

# Check CI/CD pipeline logs
# Check ECR push history

# Check deployment definition
kubectl get deployment recommendation -n otel-demo -o yaml | grep -A 5 "image:"
```

#### Step 10: Prevention
```bash
# Add image pull policy
kubectl patch deployment recommendation -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"recommendation","imagePullPolicy":"IfNotPresent"}]}}}}'

# Add liveness probe to verify health
# Add image verification in CI/CD
# Add alerts for ImagePullBackOff events
```

### Prevention Strategies
- ✅ Verify image exists before deploying
- ✅ Pin to specific image tags (not `latest`)
- ✅ Test image pull credentials regularly
- ✅ Use image scanning/signing
- ✅ Alert on ImagePullBackOff events
- ✅ Validate deployment specs in CI/CD

### Time to Resolution
**Manual investigation:** 20-45 minutes
**With automated scenario:** 5 minutes

---

## Scenario 4: CPU Throttling (Ad Service)

**What happens:** Ad service CPU usage high → Pod CPU throttled → Slow response times → Latency spike → Customer impact

### Manual SRE Investigation Steps

#### Step 1: Identify throttling
```bash
# Check CPU metrics
kubectl top pods -n otel-demo -l app=ad

# Compare requests vs limits
# If actual CPU ≈ limits, likely being throttled

# Check kernel cgroup throttling stats
kubectl exec -it <ad-pod> -n otel-demo -- cat /proc/<pid>/stat | awk '{print $43, $44, $45}'
# nr_throttled shows times pod was throttled
```

#### Step 2: Check CPU requests/limits
```bash
# What are the current limits?
kubectl get deployment ad -n otel-demo -o yaml | grep -A 10 "resources:"

# Are they too low for the workload?
# Typical CPU throttling occurs when: actual CPU > limit
```

#### Step 3: Analyze request patterns
```bash
# Query Prometheus for request rate
rate(http_requests_total{service="ad"}[5m])

# Is request rate unusually high?
# Correlate with CPU throttling events

# Check request complexity
# (some requests require more CPU than others)
```

#### Step 4: Check for CPU-intensive operations
```bash
# Profile the application
kubectl exec -it <ad-pod> -n otel-demo -- curl -s http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof

# Analyze profile
go tool pprof cpu.prof
# top20 (show top 20 CPU consumers)

# Look for:
# - Infinite loops
# - Inefficient algorithms
# - Unoptimized regex
# - Missing caches
```

#### Step 5: Check for resource contention
```bash
# Are other pods on the same node causing issues?
kubectl get pods -n otel-demo --field-selector spec.nodeName=<node> -o wide

# What's the total CPU usage on the node?
kubectl top node <node>

# Is the node at CPU capacity?
# If yes, pods will be throttled
```

#### Step 6: Check for noisy neighbors
```bash
# Are other pods consuming excessive CPU?
kubectl top pods -n otel-demo --sort-by=cpu

# Any suspicious high CPU usage?
# If so, might need to isolate or scale
```

#### Step 7: Latency correlation
```bash
# When did latency spike?
# Query: histogram_quantile(0.99, http_request_duration_seconds{service="ad"})

# Does it correlate with CPU throttling events?
kubectl get events -n otel-demo --sort-by='.lastTimestamp' | grep -i cpu

# Timeline should show:
# T+0: High request rate
# T+5: CPU throttling starts
# T+5: Latency increases
```

#### Step 8: Immediate remediation
```bash
# Option 1: Increase CPU limits
kubectl set resources deployment ad \
  -n otel-demo \
  --limits=cpu=2000m

# Option 2: Increase replicas to distribute load
kubectl scale deployment ad --replicas=5 -n otel-demo

# Option 3: Use node affinity to spread pods
kubectl patch deployment ad -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"affinity":{"podAntiAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":[{"labelSelector":{"matchExpressions":[{"key":"app","operator":"In","values":["ad"]}]},"topologyKey":"kubernetes.io/hostname"}]}}}}}}'
```

#### Step 9: Long-term optimization
```bash
# Profile and optimize hot paths
# - Cache expensive computations
# - Use more efficient algorithms
# - Parallelize where possible

# Re-baseline CPU requirements
# Load test to find realistic CPU needs

# Deploy optimized version
kubectl rollout restart deployment/ad -n otel-demo
```

#### Step 10: Verify recovery
```bash
# Monitor CPU usage
kubectl top pods -n otel-demo -l app=ad --watch

# Check latency p99
# Should return to baseline

# Verify no more throttling
kubectl get events -n otel-demo | grep -i throttle
```

### Prevention Strategies
- ✅ Right-size CPU requests based on load testing
- ✅ Monitor CPU trends over time
- ✅ Set alerts for CPU usage > 80%
- ✅ Profile applications for hot paths
- ✅ Use horizontal pod autoscaling
- ✅ Implement request throttling/rate limiting

### Time to Resolution
**Manual investigation:** 1-2 hours (optimization phase)
**With automated scenario:** 10 minutes

---

## Scenario 5: Liveness Probe Failure (Frontend)

**What happens:** Frontend liveness probe starts failing → Kubelet restarts pod repeatedly → CrashLoopBackOff → Service degraded

### Manual SRE Investigation Steps

#### Step 1: Identify the issue
```bash
# Check pod status
kubectl get pods -n otel-demo -l app=frontend

# Status: CrashLoopBackOff or just restarting frequently
# Restart count will be increasing

# Check recent restarts
kubectl get pods -n otel-demo -l app=frontend -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,AGE:.metadata.creationTimestamp
```

#### Step 2: Check liveness probe configuration
```bash
# What is the liveness probe?
kubectl get deployment frontend -n otel-demo -o yaml | grep -A 15 "livenessProbe:"

# Typical example:
# - httpGet to /health endpoint
# - initialDelaySeconds: 60
# - periodSeconds: 10
# - failureThreshold: 3
```

#### Step 3: Check probe endpoint
```bash
# Manually test the probe endpoint
kubectl port-forward -n otel-demo svc/frontend 8080:80

# In another terminal:
curl http://localhost:8080/health

# What's the response?
# Should be 200 OK for healthy
# If not 200, probe will fail
```

#### Step 4: Check application logs
```bash
# Get logs from before crash
kubectl logs -n otel-demo -l app=frontend --previous --tail=100

# Look for:
# - What caused the unhealthy state?
# - Any errors or exceptions?
# - Configuration issues?
```

#### Step 5: Check dependencies of health check
```bash
# /health endpoint usually checks:
# - Can reach database? 
# - Can reach cache?
# - Can reach other services?

# Manually verify these:
kubectl exec -it <frontend-pod> -n otel-demo -- nc -zv postgres 5432
kubectl exec -it <frontend-pod> -n otel-demo -- nc -zv redis 6379
kubectl exec -it <frontend-pod> -n otel-demo -- curl http://productcatalog:8080/health
```

#### Step 6: Check resource constraints
```bash
# Is the pod stuck waiting for resources?
kubectl describe pod <frontend-pod> -n otel-demo | grep -A 20 "Events:"

# Look for: Pending, waiting for resources

# Check node capacity
kubectl top nodes
# Is any node low on CPU/memory?
```

#### Step 7: Check recent changes
```bash
# When did this start happening?
kubectl rollout history deployment/frontend -n otel-demo

# Was there a recent deployment?
# What changed?
kubectl rollout history deployment/frontend -n otel-demo --revision=<old-revision>

# Compare to current:
kubectl get deployment frontend -n otel-demo -o yaml
```

#### Step 8: Immediate remediation
```bash
# Option 1: Rollback to previous version
kubectl rollout undo deployment/frontend -n otel-demo

# Option 2: Adjust liveness probe to be less aggressive
kubectl patch deployment frontend -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"frontend","livenessProbe":{"initialDelaySeconds":120,"periodSeconds":15,"failureThreshold":5}}]}}}}'

# Option 3: Disable liveness probe temporarily (NOT recommended long-term)
kubectl patch deployment frontend -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"frontend","livenessProbe":null}]}}}}'
```

#### Step 9: Fix the underlying issue
```bash
# If /health endpoint is failing because of a dependency:
# - Fix the dependency (database, cache, service)
# - Verify connectivity
# - Increase timeout if endpoint is slow

# Re-deploy with fix
kubectl rollout restart deployment/frontend -n otel-demo
```

#### Step 10: Verify recovery
```bash
# Monitor pod status
kubectl get pods -n otel-demo -l app=frontend --watch

# Should show: CrashLoopBackOff → Running

# Check restart count stops increasing
# Monitor for 5-10 minutes to verify stability
```

### Prevention Strategies
- ✅ Make liveness probes timeout-tolerant
- ✅ Set appropriate initialDelaySeconds
- ✅ Ensure /health endpoint is lightweight
- ✅ Make /health check only critical dependencies
- ✅ Use readiness probe for non-critical checks
- ✅ Monitor liveness probe failure rate

### Time to Resolution
**Manual investigation:** 30-60 minutes
**With automated scenario:** 5 minutes

---

## Scenario 6: Wrong Environment Variable (Checkout)

**What happens:** Checkout service gets wrong config → Can't connect to payment service → All checkouts fail → Revenue impact

### Manual SRE Investigation Steps

#### Step 1: Identify checkout failures
```bash
# Check checkout service status
kubectl get pods -n otel-demo -l app=checkout

# Check error rate
# Alert: checkout error rate > 10%

# Check logs for errors
kubectl logs -n otel-demo -l app=checkout --tail=50
# Look for: "connection refused", "timeout", "invalid configuration"
```

#### Step 2: Check environment variables
```bash
# What env vars does checkout pod have?
kubectl get pods -n otel-demo -l app=checkout -o jsonpath='{.items[0].spec.containers[0].env}'

# Or more readable:
kubectl exec -it <checkout-pod> -n otel-demo -- env | grep -i payment

# Is PAYMENT_SERVICE_URL set correctly?
# Should be: http://payment:8080 (or similar)
```

#### Step 3: Check deployment definition
```bash
# What env vars are configured?
kubectl get deployment checkout -n otel-demo -o yaml | grep -A 50 "env:"

# Is the variable spelled correctly?
# Is the value correct?
# Is it in the right format?
```

#### Step 4: Check if env var comes from ConfigMap
```bash
# Is it referenced from ConfigMap?
kubectl get configmap -n otel-demo | grep checkout

# Get the ConfigMap
kubectl get configmap checkout-config -n otel-demo -o yaml

# Is the value in ConfigMap correct?
```

#### Step 5: Verify connectivity
```bash
# Can checkout pod reach the payment service?
kubectl exec -it <checkout-pod> -n otel-demo -- nc -zv payment 8080

# Can it resolve the hostname?
kubectl exec -it <checkout-pod> -n otel-demo -- nslookup payment

# Can it curl the endpoint?
kubectl exec -it <checkout-pod> -n otel-demo -- curl http://payment:8080/health
```

#### Step 6: Check application startup logs
```bash
# Get full logs from pod creation
kubectl logs -n otel-demo -l app=checkout --timestamps=true | head -100

# Look for: Configuration loaded, connecting to payment service, status

# Does it show the wrong URL being used?
```

#### Step 7: Check recent deployments
```bash
# When did this change get deployed?
kubectl rollout history deployment/checkout -n otel-demo

# What was in the previous version?
kubectl rollout history deployment/checkout -n otel-demo --revision=<n-1>

# What changed between versions?
# Check git diff or deployment spec changes
```

#### Step 8: Immediate remediation
```bash
# Option 1: Fix and rollout
kubectl set env deployment/checkout \
  PAYMENT_SERVICE_URL=http://payment:8080 \
  -n otel-demo

# Option 2: Use ConfigMap patch
kubectl patch configmap checkout-config -n otel-demo \
  --type merge \
  -p '{"data":{"PAYMENT_SERVICE_URL":"http://payment:8080"}}'

# Force pod restart to pick up new config
kubectl rollout restart deployment/checkout -n otel-demo

# Option 3: Rollback if recent deployment caused it
kubectl rollout undo deployment/checkout -n otel-demo
```

#### Step 9: Verify fix
```bash
# Check pod status
kubectl get pods -n otel-demo -l app=checkout

# Check logs for successful connection
kubectl logs -n otel-demo -l app=checkout --tail=20 | grep -i "payment\|connected\|success"

# Test endpoint
kubectl exec -it <checkout-pod> -n otel-demo -- curl http://localhost:8080/health
```

#### Step 10: Prevent future issues
```bash
# Add validation in startup:
# - Verify all required env vars are set
# - Validate URL formats
# - Test connectivity on startup

# Add monitoring:
# - Alert on wrong configuration values
# - Check config file integrity

# Add pre-deployment checks:
# - Validate configmaps before deploying
# - Run integration tests
```

### Prevention Strategies
- ✅ Use ConfigMap/Secret validation
- ✅ Add config validation on application startup
- ✅ Use strongly-typed configuration (not strings)
- ✅ Test with wrong config in CI/CD
- ✅ Add alerts for configuration anomalies
- ✅ Implement configuration change notifications

### Time to Resolution
**Manual investigation:** 20-45 minutes
**With automated scenario:** 5 minutes

---

## Scenario 7: Database Connection Failure (Ad Service)

**What happens:** Ad service can't connect to database → Queries timeout → Request queue backs up → Service slow/unavailable

### Manual SRE Investigation Steps

#### Step 1: Identify database connectivity issue
```bash
# Check ad service logs
kubectl logs -n otel-demo -l app=ad --tail=50 | grep -i "database\|connection\|timeout"

# Alert: ad service error rate > 5%
# Alert: ad service p99 latency > 5s
```

#### Step 2: Check database pod status
```bash
# Is database pod running?
kubectl get pods -n otel-demo -l app=postgres

# Status should be Running
# If not: ImagePullBackOff, Pending, etc. indicates problem

# Check database logs
kubectl logs -n otel-demo -l app=postgres --tail=50
```

#### Step 3: Test database connectivity
```bash
# From ad pod, try to connect
kubectl exec -it <ad-pod> -n otel-demo -- nc -zv postgres 5432

# Try actual connection
kubectl exec -it <ad-pod> -n otel-demo -- psql -h postgres -U postgres -c "SELECT 1"

# If fails: timeout, connection refused, auth error?
```

#### Step 4: Check database resource capacity
```bash
# Is database CPU/memory high?
kubectl top pods -n otel-demo -l app=postgres

# Is it at limits?
kubectl get deployment postgres -n otel-demo -o yaml | grep -A 10 "resources:"

# If at limits, database is slow/unresponsive
```

#### Step 5: Check connection pool
```bash
# How many connections are active?
kubectl exec -it <postgres-pod> -n otel-demo -- psql -c "SELECT count(*) as active_connections FROM pg_stat_activity"

# What's the max allowed?
kubectl exec -it <postgres-pod> -n otel-demo -- psql -c "SHOW max_connections"

# Is there a bottleneck? (active connections ≈ max)
```

#### Step 6: Check query performance
```bash
# Are there slow queries?
kubectl exec -it <postgres-pod> -n otel-demo -- psql -c "SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10"

# Is anything causing locks?
kubectl exec -it <postgres-pod> -n otel-demo -- psql -c "SELECT * FROM pg_locks"

# Check for active transactions
kubectl exec -it <postgres-pod> -n otel-demo -- psql -c "SELECT * FROM pg_stat_activity WHERE state = 'active'"
```

#### Step 7: Check network connectivity
```bash
# Is there a network issue between ad service and database?
# Check pod networking
kubectl exec -it <ad-pod> -n otel-demo -- ping postgres

# Check DNS
kubectl exec -it <ad-pod> -n otel-demo -- nslookup postgres

# Check firewall/network policies
kubectl get networkpolicies -n otel-demo
# Is there a policy blocking ad→postgres?
```

#### Step 8: Check connection string
```bash
# What connection string is ad service using?
kubectl get deployment ad -n otel-demo -o yaml | grep -A 5 "DATABASE\|POSTGRES\|DB_"

# Is it correct? (hostname, port, username, password)
# Does postgres pod have a service? 
kubectl get svc -n otel-demo | grep postgres
```

#### Step 9: Immediate remediation
```bash
# Option 1: Scale up database
kubectl scale deployment postgres --replicas=2 -n otel-demo

# Option 2: Increase database resources
kubectl set resources deployment postgres \
  -n otel-demo \
  --limits=memory=2Gi,cpu=2000m

# Option 3: Kill idle connections to free up slots
kubectl exec -it <postgres-pod> -n otel-demo -- psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND query_start < now() - INTERVAL '5 min'"

# Option 4: Restart database pod
kubectl rollout restart deployment/postgres -n otel-demo
```

#### Step 10: Verify recovery
```bash
# Check connectivity from ad pod
kubectl exec -it <ad-pod> -n otel-demo -- psql -h postgres -U postgres -c "SELECT 1"

# Monitor ad service error rate
# Should return to near 0%

# Monitor latency
# Should return to baseline
```

### Prevention Strategies
- ✅ Monitor database connection count
- ✅ Implement connection pooling in ad service
- ✅ Set connection timeouts appropriately
- ✅ Monitor slow queries
- ✅ Implement circuit breaker for database
- ✅ Add database read replicas for scalability
- ✅ Use connection timeouts to prevent hanging

### Time to Resolution
**Manual investigation:** 45-90 minutes
**With automated scenario:** 10 minutes

---

## Scenario 8: Noisy Neighbor (Recommendation Service)

**What happens:** One service (e.g., batch job) consumes excessive resources → Other pods starved → Recommendation service slow/unavailable

### Manual SRE Investigation Steps

#### Step 1: Identify resource contention
```bash
# Check which pods are using high resources
kubectl top pods -n otel-demo --sort-by=cpu

# Which pod is consuming the most CPU/memory?
# Look for unusual spikes

# Check if it's a batch job or expected high-usage service
```

#### Step 2: Identify affected services
```bash
# Which services are degraded?
# Check error rates across all services
# Alert: recommendation service p99 latency > 5s

# Check if it correlates with high-resource consumer
```

#### Step 3: Check node allocation
```bash
# What node is the noisy neighbor on?
kubectl get pods -n otel-demo -o wide | grep <noisy-neighbor-pod>

# What other pods are on the same node?
kubectl get pods -n otel-demo --field-selector spec.nodeName=<node> -o wide

# How much CPU/memory is left for others?
kubectl top node <node>
```

#### Step 4: Check pod resource limits
```bash
# Does the noisy neighbor have appropriate limits?
kubectl get pod <noisy-pod> -n otel-demo -o yaml | grep -A 10 "resources:"

# Are limits set? Or is it unlimited?
# If unlimited, it can consume all node resources
```

#### Step 5: Identify root cause
```bash
# Is it a legitimate workload spike?
# Check logs:
kubectl logs -n otel-demo -l app=<noisy-neighbor> --tail=100 | grep -i "processing\|batch\|job"

# Is it a runaway process?
# Check CPU time:
kubectl exec -it <noisy-pod> -n otel-demo -- ps aux | head -20

# Is it expected (batch job) or unexpected (bug)?
```

#### Step 6: Check for memory leaks in noisy neighbor
```bash
# If it's a long-running process, check for memory growth
# Get memory history:
kubectl exec -it <noisy-pod> -n otel-demo -- cat /proc/meminfo

# Check for accumulating data structures
# Profile application
```

#### Step 7: Immediate remediation
```bash
# Option 1: Add resource limits to noisy neighbor
kubectl set resources deployment/<noisy-app> \
  -n otel-demo \
  --limits=cpu=1000m,memory=512Mi

# Option 2: Isolate to dedicated node (node affinity)
kubectl patch deployment <noisy-app> -n otel-demo \
  --type merge \
  -p '{"spec":{"template":{"spec":{"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"workload-type","operator":"In","values":["batch"]}]}]}}}}}}}'

# Option 3: Kill the noisy neighbor pod (if it's a runaway)
kubectl delete pod <noisy-pod> -n otel-demo

# Option 4: Scale the noisy workload if it's legitimate
# Distribute across multiple pods to avoid starving others
```

#### Step 8: Verify recovery
```bash
# Check CPU/memory consumption
kubectl top pods -n otel-demo --sort-by=cpu

# Check affected services recover
# recommendation service latency should return to baseline

# Monitor for 5-10 minutes to verify stability
```

#### Step 9: Long-term mitigation
```bash
# Set guaranteed resource requests/limits
# for all pods based on load testing

# Use QoS classes appropriately:
# - Guaranteed: critical services
# - Burstable: normal services
# - BestEffort: batch jobs (lower priority)

# Implement pod priority classes
```

#### Step 10: Monitoring
```bash
# Add alerts for:
# - Single pod consuming > 50% node CPU
# - Recommendation service latency spike
# - CPU throttling events

# Add node affinity rules to prevent co-scheduling
# incompatible workloads
```

### Prevention Strategies
- ✅ Set appropriate resource requests/limits
- ✅ Use pod affinity/anti-affinity rules
- ✅ Implement pod priority classes
- ✅ Monitor for resource contention
- ✅ Use Guaranteed QoS for critical services
- ✅ Schedule batch jobs on dedicated nodes
- ✅ Implement horizontal pod autoscaling

### Time to Resolution
**Manual investigation:** 30-60 minutes
**With automated scenario:** 10 minutes

---

## Scenario 9: Autoscaler Node Pressure

**What happens:** Heavy load → Pods pending → Cluster autoscaler detects → New nodes launch → Pods schedule → System recovers

### Manual SRE Investigation Steps

#### Step 1: Identify resource pressure
```bash
# Check for pending pods
kubectl get pods -n otel-demo --field-selector=status.phase=Pending

# How many pending? For how long?

# Check why they're pending
kubectl describe pod <pending-pod> -n otel-demo | grep -A 20 "Events:"
# Look for: "0/3 nodes are available", "Insufficient cpu", etc.
```

#### Step 2: Check cluster capacity
```bash
# Total CPU/memory available
kubectl describe nodes | grep -A 3 "Allocated resources"

# Or more clearly:
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU_REQUESTS:.status.allocatable.cpu,MEMORY_REQUESTS:.status.allocatable.memory

# How much is used?
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, cpu: .status.allocatable.cpu, memory: .status.allocatable.memory}'
```

#### Step 3: Check cluster autoscaler status
```bash
# Is autoscaler running?
kubectl get deployment -n kube-system | grep autoscaler

# Check its logs
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100 | grep -i "scaling\|pending\|asg"

# Look for: "Scaling up"
```

#### Step 4: Monitor node creation
```bash
# Are new nodes being created?
# Watch node count
kubectl get nodes --watch

# Or check AWS ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names <your-asg> \
  --region eu-west-2 \
  --query 'AutoScalingGroups[0].[DesiredCapacity,RunningInstances]'

# Should see DesiredCapacity increase
```

#### Step 5: Check node readiness
```bash
# Are new nodes joining the cluster?
kubectl get nodes -o wide

# Are they in Ready state?
# NotReady nodes can't accept pods yet

# What's the node startup time?
# From ASG launch to Ready state
```

#### Step 6: Monitor pod scheduling
```bash
# As nodes come online, pods should schedule
kubectl get pods -n otel-demo --field-selector=status.phase=Pending --watch

# How long until all pods Running?

# Check events for scheduling
kubectl get events -n otel-demo --sort-by='.lastTimestamp' | grep -i "scheduled\|binding"
```

#### Step 7: Verify autoscaler configuration
```bash
# Check autoscaler deployment
kubectl get deployment -n kube-system cluster-autoscaler -o yaml | grep -A 20 "args:"

# Look for:
# - scale-down-enabled: true/false
# - scale-down-delay-after-add: 10m (default)
# - skip-nodes-with-local-storage: true/false
```

#### Step 8: Check performance impact
```bash
# What's the latency during scaling?
# Query: histogram_quantile(0.99, http_request_duration_seconds)
# Should see spike during scaling, then recover

# Check error rate
# Should stay low even during scaling

# Check if requests are queued during scaling
# Should recover as new capacity comes online
```

#### Step 9: Monitor scale-down
```bash
# After peak load passes, autoscaler should scale down
# Check in 10+ minutes (default cooldown)

kubectl get nodes --watch
# Should see node count decrease

# Check which nodes are removed
kubectl get events | grep -i "removing\|cordoning"
```

#### Step 10: Post-scaling analysis
```bash
# How long was scaling taking?
# Acceptable: 2-5 minutes

# How long until pods scheduled?
# Acceptable: < 3 minutes

# Were any requests dropped?
# Should be minimal

# What could have been better?
# - Faster node startup?
# - Pre-warmed node pool?
# - Better pod packing?
```

### Prevention Strategies
- ✅ Set appropriate pod requests for accurate scheduling
- ✅ Use cluster autoscaler with reasonable limits
- ✅ Pre-provision some capacity for quick scaling
- ✅ Use horizontal pod autoscaling + cluster autoscaling
- ✅ Monitor autoscaler performance
- ✅ Set appropriate scale-down delays
- ✅ Avoid local storage (prevents node removal)

### Time to Resolution
**Manual investigation:** N/A (this is capacity planning)
**With automated scenario:** 10 minutes (to validate autoscaling works)

---

## Scenario 10: Combined App + Infrastructure Failure

**What happens:** Application fails (slow productcatalog) + infrastructure fails (node memory pressure) → Cascade → Multiple layers of failure → Complex root cause

### Manual SRE Investigation Steps

#### Step 1: Multi-layer diagnosis
```bash
# Layer 1: Application failures
kubectl logs -n otel-demo -l app=productcatalog --tail=50 | grep -i "error\|timeout"
kubectl logs -n otel-demo -l app=frontend --tail=50 | grep -i "error\|timeout"

# Layer 2: Infrastructure failures
kubectl get pods -n otel-demo -o wide | grep -i "evicted\|oomkilled"
kubectl get events -n otel-demo | grep -i "evict\|oom\|pressure"

# Layer 3: Resource pressure
kubectl top nodes
kubectl describe nodes | grep -A 5 "MemoryPressure\|DiskPressure"
```

#### Step 2: Timeline correlation
```bash
# Get event timeline
kubectl get events -n otel-demo --sort-by='.lastTimestamp' -o wide

# Manual timeline construction:
# T+0: What happened first?
# T+30: What cascaded from that?
# T+60: What was the final state?

# Which events are related?
# Look for time correlations
```

#### Step 3: Application layer diagnosis
```bash
# Step 3a: Slow productcatalog
# Check productcatalog logs
kubectl logs -n otel-demo -l app=productcatalog --tail=50

# Check if it has database connectivity
kubectl exec -it <productcatalog-pod> -n otel-demo -- psql -h postgres -U postgres -c "SELECT 1"

# Check latency
# Query OTEL traces for productcatalog latency
```

#### Step 4: Infrastructure layer diagnosis
```bash
# Step 4a: Node memory pressure
kubectl top nodes
# Is memory > 80%? > 90%?

# Step 4b: Pod evictions
kubectl get events -n otel-demo | grep -i evict
# Which pods are being evicted?
# Why? (memory pressure, disk pressure, etc.)

# Step 4c: OOMKilled pods
kubectl get pods -n otel-demo -o yaml | grep -i oomkilled
```

#### Step 5: Causal analysis
```bash
# Q1: What was the initial failure?
# - Application slow? (productcatalog DB timeout)
# - Infrastructure pressure? (node memory)
# - Which came first?

# Q2: How did it cascade?
# Timeline:
# 1. Application slow → buffered requests
# 2. Buffered requests use memory
# 3. Node memory pressure increases
# 4. Kubelet evicts pods (lowest QoS first)
# 5. Frontend might get evicted
# 6. Service completely unavailable

# Q3: Which layer to fix first?
# Both! Fix application latency AND allocate more memory
```

#### Step 6: Identify blast radius
```bash
# Which services were affected?
# - productcatalog: yes (source)
# - frontend: yes (blocked on catalog)
# - checkout: maybe (depends on frontend)
# - cart: maybe (depends on frontend)

# How many users affected?
# Depends on traffic patterns

# How long was the outage?
# From first failure until recovery
```

#### Step 7: OTEL distributed tracing analysis
```bash
# Open Jaeger UI
# Filter: service=frontend, time=incident window

# Look for trace showing:
# 1. Frontend → productcatalog call
# 2. Productcatalog latency (high)
# 3. Span duration >> normal
# 4. Possibly timeout or error

# Look for multiple attempts/retries
# Retry backoff pattern?

# What was the full latency from user to backend?
```

#### Step 8: Metric correlation
```bash
# In Prometheus, correlate:
# - productcatalog_latency (spike) + 
# - node_memory_usage (high) +
# - pod_eviction_count (increase) =
# Multiple independent failures

# Timeline should show:
# T+0: productcatalog latency spike
# T+5: node_memory_usage > 90%
# T+10: pod_eviction_count increases
# T+15: frontend pod evicted
# T+20: service unavailable
```

#### Step 9: Remediation strategy
```bash
# Fix Application Layer:
# 1. Fix productcatalog DB connection timeout
# 2. Restart productcatalog service
# 3. Verify latency returns to normal

# Fix Infrastructure Layer:
# 1. Delete evicted pods (will restart)
# 2. Increase node memory OR add new node
# 3. Verify memory pressure decreases

# Coordinate fixes:
# Do application fix first (prevents memory buildup)
# Then infrastructure fix if needed
```

#### Step 10: Post-incident analysis
```bash
# What was the root cause?
# - Application: slow queries in productcatalog
# - Infrastructure: insufficient node memory

# What broke the isolation?
# - Frontend buffered requests (consumed memory)
# - Memory pressure affected all pods (not isolated)

# How to prevent:
# 1. Fix productcatalog queries (SQL optimization)
# 2. Set memory requests/limits to prevent starvation
# 3. Add circuit breaker to frontend
# 4. Implement pod disruption budget
# 5. Monitor memory trends
```

### Prevention Strategies
- ✅ Fix application inefficiencies (slow queries, etc.)
- ✅ Right-size resource requests/limits
- ✅ Implement circuit breakers
- ✅ Use pod disruption budgets
- ✅ Monitor for cascading failures
- ✅ Implement bulkhead pattern (service isolation)
- ✅ Add distributed tracing for visibility

### Time to Resolution
**Manual investigation:** 2-4 hours (multi-layer diagnosis)
**With automated scenario:** 15 minutes (learn the pattern)

---

## Summary: SRE Skills Demonstrated

| Skill | Scenario | Time |
|-------|----------|------|
| Service troubleshooting | Cascading failure | 30-60m |
| Memory debugging | OOMKilled | 1-2h |
| Image/registry issues | Image pull failure | 20-45m |
| Performance tuning | CPU throttling | 1-2h |
| Health check issues | Liveness probe | 30-60m |
| Configuration management | Wrong env var | 20-45m |
| Database debugging | DB connection | 45-90m |
| Resource isolation | Noisy neighbor | 30-60m |
| Cluster scaling | Autoscaler | ongoing |
| Multi-layer diagnosis | Combined failure | 2-4h |

## Value of Automated Scenarios

With these scenarios, an SRE can practice all these skills in **minutes** instead of waiting for them to happen in production (which might take **hours** each time).

**Benefits:**
- ✅ Practice troubleshooting under controlled conditions
- ✅ Build muscle memory for common issues
- ✅ Test runbooks before production incident
- ✅ Demonstrate root cause analysis to team
- ✅ Build confidence in infrastructure
- ✅ Identify systemic weaknesses

Each scenario is a "failure resilience drill" - like fire drills for infrastructure!
