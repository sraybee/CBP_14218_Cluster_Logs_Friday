# PreProd Controller Error Analysis Report
**Date:** March 20, 2026
**Purpose:** Pre-production error analysis before promoting Tekton v1.10.0 to Production
**Ticket:** CBP-14218

---

## Executive Summary

Analyzed logs from all 8 CloudBees Automation Controller pods across 4 PreProd clusters. Found **3 error types**, with **1 CRITICAL issue** that requires attention before production deployment.

---

## Error Summary by Cluster

| Cluster | Status | Controller Version | Total Errors | Critical | Auth Errors |
|---------|--------|-------------------|--------------|----------|-------------|
| us-east-1-blue | **ACTIVE** | 0.0.1725 | 8 | **7** | 1 |
| us-east-1-gree | STANDBY | 0.0.1734 | 919 | 0 | 919 |
| us-west-2-blue | STANDBY | 0.0.1734 | 1,306 | 0 | 1,306 |
| us-west-2-gree | **ACTIVE** | 0.0.1725 | 905 | 0 | 905 |
| **TOTAL** | | | **3,138** | **7** | **3,131** |

---

## 🚨 CRITICAL ISSUE #1: DisableAffinityAssistant Field Error

### Details
- **Severity:** HIGH - Could cause workflow failures in production
- **Affected Cluster:** tekton-preprod-us-east-1-blue (ACTIVE)
- **Occurrences:** 7 errors over ~1.5 hours (14:01 - 15:41)
- **Controller Version:** 0.0.1725
- **Tekton Version:** v1.10.0

### Error Message
```
ERROR Reconciler error {"controller": "metaPipelineStatusController",
"TaskRun": {"name":"dispatch-dispatch","namespace":"event--acaaecfa5548863b0571fe25c45a9124733b88f839f13c527b17cb24"},
"error": "admission webhook \"webhook.pipeline.tekton.dev\" denied the request:
mutation failed: cannot decode incoming old object: json: unknown field \"DisableAffinityAssistant\""}
```

### Root Cause
The `DisableAffinityAssistant` field was **removed in Tekton v1.0.0**. This error occurs when:
1. An old TaskRun object from Tekton v0.62.9 (or earlier) exists in the cluster
2. The Controller tries to update/reconcile that TaskRun
3. Tekton v1.10.0 admission webhook rejects it because the field no longer exists

### Impact Analysis
- **Current Impact:** Reconciliation failures on ONE specific TaskRun (`dispatch-dispatch` in namespace `event--acaaecfa5548863b0571fe25c45a9124733b88f839f13c527b17cb24`)
- **Pattern:** Repeated attempts to reconcile the same stuck object every ~15-17 minutes
- **Workflow Impact:** This specific workflow/event namespace appears stuck
- **Broader Impact:** No other TaskRuns affected (only 1 out of potentially hundreds)

### Production Risk Assessment
**MEDIUM-HIGH RISK:**
- ✅ **Good:** Only affects pre-existing TaskRuns with deprecated fields
- ✅ **Good:** New workflows created on v1.10.0 won't have this issue
- ⚠️ **Concern:** Any long-running workflows from v0.62.9 era could hit this
- ⚠️ **Concern:** Workflows that span the upgrade window could fail
- ❌ **Bad:** No automatic cleanup mechanism for stuck resources

### Recommended Actions

#### Option 1: Clean Up Stuck Resources (RECOMMENDED)
```bash
# Delete the stuck TaskRun
kubectl delete taskrun dispatch-dispatch \
  -n event--acaaecfa5548863b0571fe25c45a9124733b88f839f13c527b17cb24 \
  --kubeconfig /tmp/kube-east-blue

# Find and clean other potential stuck resources
kubectl get taskruns --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.taskSpec.params[].name == "DisableAffinityAssistant") | .metadata.namespace + "/" + .metadata.name'
```

#### Option 2: Update Controller to Handle Legacy Fields
- Check if newer controller version (0.0.1734) handles this gracefully
- Test on STANDBY clusters first

#### Option 3: Migration Script
Create a pre-upgrade script to clean/migrate old TaskRuns before Tekton upgrade

### Verification Steps
1. ✅ **Identify affected namespace:** `event--acaaecfa5548863b0571fe25c45a9124733b88f839f13c527b17cb24`
2. ⏳ **Check if workflow completed:** Query TaskRun status
3. ⏳ **Clean up if stuck:** Delete TaskRun if workflow is abandoned/old
4. ⏳ **Monitor for recurrence:** Ensure no new errors after cleanup
5. ⏳ **Document for production:** Add to production runbook

---

## ⚠️ ISSUE #2: IAM/ECR Authentication Errors

### Details
- **Severity:** MEDIUM - Indicates configuration issue, not Tekton v1.10.0 issue
- **Affected Clusters:** ALL clusters (3,131 total errors)
- **Most Affected:** us-west-2-blue (STANDBY) - 1,306 errors
- **Pattern:** Continuous failures across all clusters

### Error Message
```
ERROR Reconciler error {"controller": "imagepullsecretsrequest",
"error": "could not retrieve ECR authorization token: operation error ECR: GetAuthorizationToken,
get identity: get credentials: failed to retrieve credentials,
operation error STS: AssumeRoleWithWebIdentity, https response error StatusCode: 403,
api error AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity"}
```

### Root Cause
IAM Role for Service Account (IRSA) configuration issue:
- Service account cannot assume the IAM role needed for ECR access
- Could be:
  - Trust relationship misconfiguration
  - Service account annotations incorrect
  - OIDC provider issue
  - Temporary AWS credential/permission issue

### Impact Analysis
- **Current Impact:** Workflows requiring private ECR images may fail to pull images
- **Tekton Relation:** **NOT RELATED** to Tekton v1.10.0 upgrade
- **Timeline:** Appears to be pre-existing or concurrent issue
- **Scope:** Affects image pull secret creation for ECR

### Production Risk Assessment
**MEDIUM RISK (Not Tekton-related):**
- ⚠️ **Pre-existing issue** - Not caused by Tekton upgrade
- ⚠️ **High volume** - 3,131 errors suggest many workflows affected
- ✅ **Isolated to ECR** - Non-ECR workflows unaffected
- ⚠️ **Should be investigated** - But separately from Tekton upgrade

### Recommended Actions
1. **Verify IRSA Configuration:**
   ```bash
   # Check service account annotations
   kubectl get sa -n cloudbees-automation-controller -o yaml | grep eks.amazonaws.com/role-arn

   # Verify IAM role trust policy
   aws iam get-role --role-name <controller-role-name> --profile saas-pp-dev
   ```

2. **Check OIDC Provider:**
   ```bash
   # Verify OIDC provider exists for cluster
   aws iam list-open-id-connect-providers --profile saas-pp-dev
   ```

3. **Test ECR Access:**
   ```bash
   # Create test pod to verify ECR pull
   kubectl run test-ecr --image=020229604682.dkr.ecr.us-east-1.amazonaws.com/test:latest
   ```

4. **Review Recent IAM Changes:**
   - Check CloudTrail for IAM policy/role modifications
   - Verify no AWS organization SCP changes

### Verification
- ⏳ **Separate investigation** - Track under different ticket
- ⏳ **Does not block Tekton upgrade** - But should be resolved
- ⏳ **May affect production** - If not fixed before prod deployment

---

## ℹ️ ISSUE #3: OIDC Token Exchange Error

### Details
- **Severity:** LOW - Single occurrence
- **Affected Cluster:** us-east-1-blue (ACTIVE)
- **Occurrences:** 1 error at 14:15:54Z
- **Likely Related:** To IAM/ECR auth issues above

### Error Message
```
ERROR Reconciler error {"controller": "imagepullsecretsrequest",
"error": "could not exchange platform identity token for OIDC token:
[POST /token-exchange/oidc-id-token/audience/{audience}][401]
tokenExchangeServiceGetOidcIdTokenUnauthorized {\"code\":16,\"message\":\"unauthenticated\"}"}
```

### Root Cause
Platform identity token exchange failure - authentication issue with token service

### Impact Analysis
- **Current Impact:** Minimal - only 1 occurrence
- **Relation:** Part of the broader auth issue (Issue #2)

### Recommended Actions
- Monitor as part of IAM/ECR investigation (Issue #2)
- No separate action needed

---

## 📊 Missing Expected Logs

### CBP-14218 Debug Logs: NOT FOUND
- **Expected:** Debug logs for `LastTransitionTime` verification
- **Search Pattern:** "CBP-14218 Log, Remove Later, Subhradeep"
- **Result:** 0 occurrences across all logs

### Possible Explanations
1. Debug log was added in controller v0.0.1734 (STANDBY clusters)
2. We collected logs from v0.0.1725 (ACTIVE clusters primarily)
3. Debug log trigger conditions haven't been met yet
4. Need to collect logs from newer controller versions

### Recommendation
- Collect fresh logs from STANDBY clusters (v0.0.1734) to verify debug log
- Or wait for next controller deployment to ACTIVE clusters

---

## 🎯 Production Deployment Recommendations

### 🚨 MUST FIX Before Production
1. **Resolve DisableAffinityAssistant errors:**
   - Delete stuck TaskRun in us-east-1-blue
   - Scan ALL clusters for similar stuck resources
   - Create cleanup runbook for production

### ⚠️ SHOULD INVESTIGATE (Can proceed with caution)
2. **IAM/ECR Authentication Issues:**
   - Open separate ticket for IRSA investigation
   - Does NOT block Tekton upgrade
   - BUT could cause production workflow failures (unrelated to Tekton)

### ✅ Safe to Proceed
3. **No Tekton v1.10.0 breaking changes detected** (except legacy field cleanup)
4. **No panics or fatal errors**
5. **Controllers are stable and running**

---

## 📋 Production Deployment Checklist

### Pre-Deployment
- [ ] Clean up stuck TaskRuns with `DisableAffinityAssistant` field
- [ ] Verify no old PipelineRuns with deprecated fields
- [ ] Document IAM/ECR issue for production monitoring
- [ ] Create rollback plan if DisableAffinityAssistant issues arise
- [ ] Test cleanup script in PreProd STANDBY cluster first

### During Deployment
- [ ] Monitor for DisableAffinityAssistant errors immediately after upgrade
- [ ] Have cleanup script ready to execute if needed
- [ ] Watch for workflows that span the upgrade window

### Post-Deployment
- [ ] Verify no DisableAffinityAssistant errors in first 2 hours
- [ ] Monitor IAM/ECR errors (track separately)
- [ ] Confirm new workflows run successfully
- [ ] Keep blue clusters (old version) for 4-day rollback window

---

## 📁 Log Collection Details

**Collection Time:** March 20, 2026 21:11-21:42 IST
**Total Pods:** 8
**Total Log Size:** ~25MB compressed
**Clusters Analyzed:** 4 (2 ACTIVE, 2 STANDBY)
**Time Period Covered:** ~6 hours (ACTIVE clusters), ~1 hour (STANDBY clusters)

---

## Conclusion

**GO/NO-GO Decision: 🟡 GO WITH CAUTION**

**Recommendation:** Proceed to production with the following conditions:
1. ✅ Clean up stuck TaskRuns with `DisableAffinityAssistant` field BEFORE upgrade
2. ✅ Monitor closely for first 2 hours after production deployment
3. ⚠️ Track IAM/ECR auth errors separately (not a blocker, but needs resolution)
4. ✅ Have rollback plan ready (blue/green strategy supports this)

**Overall Risk Level:** MEDIUM
**Primary Risk:** Legacy field compatibility (manageable with cleanup)
**Secondary Risk:** IAM/ECR auth (pre-existing, not Tekton-related)

---

**Prepared by:** Claude Code Analysis
**Date:** March 20, 2026
**Report Location:** `/Users/sray/CBP_14218/controller-logs-preprod/ERROR_ANALYSIS_REPORT.md`
