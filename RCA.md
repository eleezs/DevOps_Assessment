# Root Cause Analysis: Sycamore API Pod CrashLoopBackOff

## Executive Summary
The pod entered a `CrashLoopBackOff` state due to an `OOMKilled` error caused by a memory leak in the application code and insufficient memory allocation.

---

## Investigation Process

### 1. Initial Diagnosis
**Command used:**
```bash
kubectl get pods
```

**Output:**
```
NAME                            READY   STATUS      RESTARTS   AGE
sycamore-api-65dc488ff4-vqtgs   0/1     OOMKilled   1 (4s ago) 2m7s
```

**Finding:** Pod status shows `OOMKilled` indicating Out of Memory termination.

---

### 2. Detailed Pod Inspection
**Command used:**
```bash
kubectl describe pod sycamore-api-65dc488ff4-vqtgs
```

**Key findings:**
- Last State: Terminated with Reason: `OOMKilled`
- Exit Code: 137 (killed by system)
- Memory limit: 64Mi
- Memory request: 32Mi

---

### 3. Log Analysis
**Command used:**
```bash
kubectl logs sycamore-api-65dc488ff4-vqtgs
```

**Finding:** No application logs, indicating the process was killed before proper startup.

---

## Root Causes Identified

### Issue 1: Memory Leak in Application Code
**Problem:**
```javascript
let arr=[]; 
setInterval(() => { 
  arr.push(new Array(1000000).fill('data')) 
}, 100)
```

- Creates an infinite memory leak
- Allocates ~8MB every 100ms
- Array grows unbounded without cleanup
- Exceeds 64Mi limit in less than 1 second

### Issue 2: Insufficient Memory Allocation
- Memory limit: 64Mi (too low for any Node.js application)
- Memory request: 32Mi (insufficient for Node.js runtime)

### Issue 3: Wrong Node.js Version
- Manifest uses: `node:18-alpine`
- Application built with: `node:14-alpine` (per Dockerfile.fix)
- Version mismatch can cause compatibility issues

### Issue 4: Missing Health Checks
- No liveness or readiness probes
- Kubernetes cannot detect application health
- No graceful restart mechanism

---

## Solution Implemented

### 1. Fixed Application Code
Replaced memory leak with proper Express.js application:
```javascript
const express = require('express');
const app = express();
app.get('/', (req, res) => {
  res.json({ status: 'Healthy' });
});
app.listen(3000);
```

### 2. Increased Memory Limits
- Memory limit: 256Mi (adequate for Node.js + Express)
- Memory request: 128Mi (realistic baseline)
- CPU limit: 200m
- CPU request: 100m

### 3. Corrected Node.js Version
- Changed from `node:18-alpine` to `node:14-alpine`
- Matches application's Dockerfile.fix specification

### 4. Added Health Probes
- Liveness probe: HTTP GET on port 3000
- Readiness probe: HTTP GET on port 3000
- Enables Kubernetes to monitor application health

---

## Verification

**Commands to verify fix:**
```bash
# Apply fixed manifest
kubectl apply -f fixed-manifest.yaml

# Check pod status
kubectl get pods

# Verify pod is running
kubectl describe pod sycamore-api-7d8bc9f7cc-plj7l 

# Check logs
kubectl logs sycamore-api-7d8bc9f7cc-plj7l 

# Monitor resource usage
kubectl top pod sycamore-api-7d8bc9f7cc-plj7l 
```

**Expected result:**
```
NAME                           READY   STATUS    RESTARTS   AGE
sycamore-api-7d8bc9f7cc-plj7l  1/1     Running   0          30s
```

---

## Lessons Learned

1. **Memory Management:** Always profile applications to set appropriate resource limits
2. **Health Checks:** Implement probes for production workloads
3. **Version Consistency:** Ensure container images match application requirements
4. **Monitoring:** Use `kubectl top` and `kubectl describe` for resource diagnostics
5. **Code Quality:** Avoid unbounded data structures and memory leaks

---

## Additional Recommendations

1. **Implement monitoring:** Add Prometheus metrics for memory usage
2. **Set up alerts:** Configure alerts for OOMKilled events
3. **Use HPA:** Consider Horizontal Pod Autoscaler for traffic scaling
4. **Resource quotas:** Define namespace-level resource limits
5. **Load testing:** Test memory usage under realistic load conditions
