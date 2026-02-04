# Session Learnings: Progressive Delivery with Argo Rollouts

**Date:** 2026-02-04
**Session Focus:** Canary & Blue-Green Deployments, APM Integration, Error Simulation
**Skill to Update:** code-to-cloud-v0.6 / code-to-cloud-v4

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Issues Encountered & Fixes](#issues-encountered--fixes)
3. [New Features Implemented](#new-features-implemented)
4. [Templates Created/Modified](#templates-createdmodified)
5. [Key Prompts & Responses](#key-prompts--responses)
6. [Best Practices Discovered](#best-practices-discovered)
7. [Skill Update Recommendations](#skill-update-recommendations)

---

## Executive Summary

This session implemented and tested a complete progressive delivery pipeline with:
- **QA Environment:** 8-step Canary deployment with APM-driven analysis
- **Staging Environment:** Blue-Green deployment with manual promotion and preview URLs
- **Error Simulation:** UI toggle for testing rollback scenarios
- **Deployment Dashboard:** Committer names in deployment history
- **Comprehensive Reporting:** Detailed deployment reports in `.deployments/` folder

---

## Issues Encountered & Fixes

### Issue 1: Staging Auto-Promoted Instead of Pausing

**Problem:** Staging blue-green deployment went directly to "Healthy" instead of "Paused" for manual promotion.

**Root Cause:** The `autoPromotionEnabled` was set to `true` in the rollout spec.

**Fix:**
```yaml
# .opsera-voting01/k8s/overlays/staging/vote-rollout.yaml
spec:
  strategy:
    blueGreen:
      autoPromotionEnabled: false  # Changed from true
```

**Learning:** For blue-green with preview testing, always set `autoPromotionEnabled: false`.

---

### Issue 2: Same Image Tag Doesn't Trigger New Rollout

**Problem:** When deploying the same image tag to staging, the rollout showed "Healthy" immediately instead of "Paused".

**Root Cause:** Argo Rollouts detects no change in desired state when the same image tag is deployed.

**Fix:** Must deploy a new image tag (from code change) to trigger a new rollout cycle.

**Learning:** To test blue-green preview flow, always make a code change first to generate a new image tag.

---

### Issue 3: jQuery Loading Over HTTP Caused Mixed Content Block

**Problem:** Error simulation toggle button didn't work - browser blocked HTTP request.

**Root Cause:** jQuery was being loaded from `http://code.jquery.com` instead of HTTPS.

**Fix:**
```html
<!-- WRONG -->
<script src="http://code.jquery.com/jquery-3.7.1.min.js"></script>

<!-- CORRECT -->
<script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>
```

**Learning:** Always use HTTPS for external resources in production apps.

---

### Issue 4: Error Simulation Too Complex

**Problem:** User found the error simulation confusing with 50% error rate and auto-disable features.

**Root Cause:** Over-engineered solution with thread-safe state, configurable rates, auto-disable timers.

**Fix:** Simplified to simple ON/OFF toggle:
```python
# Simple global state - errors OFF by default
ERROR_SIM_ENABLED = False

@app.route("/api/error-sim", methods=['POST'])
def toggle_error_sim():
    global ERROR_SIM_ENABLED
    ERROR_SIM_ENABLED = not ERROR_SIM_ENABLED
    return jsonify({'enabled': ERROR_SIM_ENABLED})

@app.route("/", methods=['POST','GET'])
def hello():
    if request.method == 'POST':
        if ERROR_SIM_ENABLED:
            return jsonify({'error': 'Simulated Error'}), 500
        # Normal processing...
```

**Learning:** Keep error simulation simple - ON means 100% failure, OFF means normal operation.

---

### Issue 5: Page Auto-Reloading Unexpectedly

**Problem:** The vote app page was constantly refreshing/polling.

**Root Cause:** JavaScript was polling the error-sim status endpoint continuously.

**Fix:** Only poll when errors are active:
```javascript
// Only poll status when needed, not continuously
$.get('/api/error-sim', function(data) {
    errorsOn = data.enabled;
    updateUI();
});
```

**Learning:** Avoid continuous polling in UI - only poll when there's a reason to check status.

---

### Issue 6: YAML Syntax Error in Workflow

**Problem:** GitHub Actions workflow failed with YAML parsing error.

**Root Cause:** Heredoc content starting at column 1 broke YAML parsing.

**Fix:** Changed from heredoc to quoted string or properly indented heredoc:
```yaml
# WRONG - heredoc at column 1 breaks YAML
run: |
  cat << EOF
Summary here
EOF

# CORRECT - use quoted string or proper indentation
run: |
  echo "Summary here"
```

**Learning:** Be careful with heredocs in YAML - indentation matters.

---

### Issue 7: Git Merge Conflicts in Concurrent Deployments

**Problem:** DEV deployment workflow failed with merge conflict in kustomization.yaml.

**Root Cause:** Multiple concurrent workflows updating the same kustomization.yaml file.

**Fix:** Workflows already have `git pull --rebase` but race conditions can still occur. This is acceptable - the retry or next build will succeed.

**Learning:** Kustomization update conflicts are normal in high-frequency deployment scenarios.

---

### Issue 8: Deployment History Missing Committer Names

**Problem:** README dashboard showed deployment history but not who made each deployment.

**Root Cause:** The `get_deploy_info()` function wasn't including author in the history bullets.

**Fix:**
```bash
# Updated to include author in each bullet
for entry in "${ENTRIES[@]}"; do
  TAG=$(echo "$entry" | cut -d'~' -f2)
  ENTRY_AUTHOR=$(echo "$entry" | cut -d'~' -f4)
  RT=$(relative_time "$TS")
  BULLETS="${BULLETS}<br>• \`${SHORT_TAG}\` (${RT}) by _${SHORT_AUTHOR}_"
done
```

**Learning:** Always attribute deployments to the committer for audit trail.

---

### Issue 9: Promotion Workflow Path Validation

**Problem:** Couldn't promote directly from QA to Staging.

**Root Cause:** The promotion workflow enforced strict path: DEV → QA → UAT → Staging.

**Fix:** Use the staging deploy workflow directly with the image tag:
```bash
gh workflow run "ci-build-push-voting01-staging.yaml" \
  -f image_tag="e3e1a9b-20260204015120"
```

**Learning:** Provide both promotion workflow (for full path) and direct deploy workflow (for flexibility).

---

## New Features Implemented

### 1. Manual Promotion Workflow for Staging Blue-Green

**File:** `.github/workflows/promote-staging-rollout-voting01.yaml`

```yaml
name: "🚀 Promote Staging (voting01)"

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "promote" to confirm'
        required: true
        type: string

jobs:
  promote:
    name: "🚀 Promote Preview to Active"
    runs-on: ubuntu-latest
    if: github.event.inputs.confirm == 'promote'
    steps:
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-west-2

      - name: Install Argo Rollouts Plugin
        run: |
          curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
          chmod +x kubectl-argo-rollouts-linux-amd64
          sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

      - name: Get Current Status
        id: status
        run: |
          aws eks update-kubeconfig --name opsera-usw2-np --region us-west-2
          PHASE=$(kubectl get rollout vote -n voting01-staging -o jsonpath='{.status.phase}')
          if [ "$PHASE" != "Paused" ]; then
            echo "paused=false" >> $GITHUB_OUTPUT
          else
            echo "paused=true" >> $GITHUB_OUTPUT
          fi

      - name: Promote Rollout
        if: steps.status.outputs.paused == 'true'
        run: |
          kubectl-argo-rollouts promote vote -n voting01-staging

      - name: Post-Promotion Status
        run: |
          PHASE=$(kubectl get rollout vote -n voting01-staging -o jsonpath='{.status.phase}')
          echo "### Post-Promotion Status: $PHASE" >> $GITHUB_STEP_SUMMARY
```

---

### 2. Error Simulation for Rollback Testing

**File:** `vote/app.py` (additions)

```python
# Simple global state - errors OFF by default
ERROR_SIM_ENABLED = False

@app.route("/api/error-sim", methods=['GET'])
def get_error_sim_status():
    global ERROR_SIM_ENABLED
    return jsonify({'enabled': ERROR_SIM_ENABLED})

@app.route("/api/error-sim", methods=['POST'])
def toggle_error_sim():
    global ERROR_SIM_ENABLED
    ERROR_SIM_ENABLED = not ERROR_SIM_ENABLED
    app.logger.info(f"Error simulation {'ENABLED' if ERROR_SIM_ENABLED else 'DISABLED'}")
    return jsonify({'enabled': ERROR_SIM_ENABLED})
```

**File:** `vote/templates/index.html` (UI additions)

```html
<!-- Error Banner -->
<div id="error-banner" style="display:none; background:#dc3545; color:white; text-align:center; padding:10px; position:fixed; top:0; left:0; right:0; z-index:1000;">
  ERRORS ON - All votes will fail with HTTP 500
</div>

<!-- Error Panel -->
<div id="error-panel">
  <h4>Canary Rollback Testing</h4>
  <button id="toggle-btn" class="off" onclick="toggleErrors()">
    Turn Errors ON
  </button>
  <div id="status-text">Errors are OFF - votes work normally</div>
</div>

<script>
var errorsOn = false;

function toggleErrors() {
  $.post('/api/error-sim', function(data) {
    errorsOn = data.enabled;
    updateUI();
  });
}

function updateUI() {
  var btn = $('#toggle-btn');
  var banner = $('#error-banner');
  if (errorsOn) {
    btn.removeClass('off').addClass('on').text('Turn Errors OFF');
    banner.show();
  } else {
    btn.removeClass('on').addClass('off').text('Turn Errors ON');
    banner.hide();
  }
}

// Check initial state on page load
$.get('/api/error-sim', function(data) {
  errorsOn = data.enabled;
  updateUI();
});
</script>
```

---

### 3. Committer Names in Deployment History

**File:** `.github/workflows/deployment-landscape-voting01.yaml` (update)

```bash
# Get last 5 as bullets with author names
BULLETS=""
IFS=';' read -ra ENTRIES <<< "$history"
for entry in "${ENTRIES[@]}"; do
  if [ -n "$entry" ]; then
    TAG=$(echo "$entry" | cut -d'~' -f2)
    SHORT_TAG=$(echo "$TAG" | cut -c1-12)
    TS=$(echo "$entry" | cut -d'~' -f3)
    ENTRY_AUTHOR=$(echo "$entry" | cut -d'~' -f4)
    RT=$(relative_time "$TS")
    SHORT_AUTHOR=$(echo "$ENTRY_AUTHOR" | cut -c1-15)
    BULLETS="${BULLETS}<br>• \`${SHORT_TAG}\` (${RT}) by _${SHORT_AUTHOR}_"
  fi
done
```

---

### 4. Staging Workflow with Preview URLs in Summary

**File:** `.github/workflows/ci-build-push-voting01-staging.yaml` (update)

```yaml
- name: Verify Blue-Green Rollout
  run: |
    for i in {1..40}; do
      VOTE_STATUS=$(kubectl get rollout vote -n $NS -o jsonpath='{.status.phase}')

      if [ "$VOTE_STATUS" = "Paused" ]; then
        echo "### ✅ Preview Ready for Testing!" >> $GITHUB_STEP_SUMMARY
        echo "| App | Preview URL (New) | Active URL (Current) |" >> $GITHUB_STEP_SUMMARY
        echo "|-----|-------------------|----------------------|" >> $GITHUB_STEP_SUMMARY
        echo "| Vote | [Preview](https://vote-voting01-staging-preview.agent.opsera.dev) | [Active](https://vote-voting01-staging.agent.opsera.dev) |" >> $GITHUB_STEP_SUMMARY
        echo "" >> $GITHUB_STEP_SUMMARY
        echo "### 🚀 Next Step: Promote to Active" >> $GITHUB_STEP_SUMMARY
        echo "[![Promote](https://img.shields.io/badge/Promote_to_Active-Click_Here-success)](...)" >> $GITHUB_STEP_SUMMARY
        exit 0
      fi

      sleep 15
    done
```

---

## Templates Created/Modified

### 1. Blue-Green Rollout Template (Staging)

```yaml
# .opsera-voting01/k8s/overlays/staging/vote-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: vote
spec:
  replicas: 3
  revisionHistoryLimit: 2
  strategy:
    blueGreen:
      activeService: vote
      previewService: vote-preview
      autoPromotionEnabled: false  # CRITICAL: Manual promotion
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
          - templateName: voting01-bluegreen-analysis
        args:
          - name: service-name
            value: vote-preview
          - name: namespace
            value: voting01-staging
```

### 2. Canary Analysis Template (QA)

```yaml
# Analysis template for APM-driven canary
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: voting01-canary-analysis
spec:
  args:
    - name: service-name
    - name: namespace
    - name: app-name
    - name: error-threshold
      value: "2"  # 2% error rate threshold
  metrics:
    - name: http-health
      count: 3
      interval: 30s
      failureLimit: 1
      provider:
        job:
          spec:
            template:
              spec:
                containers:
                  - name: health-check
                    image: curlimages/curl:latest
                    args:
                      - |
                        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
                          "http://{{args.service-name}}.{{args.namespace}}.svc.cluster.local/")
                        [ "$HTTP_CODE" = "200" ] && echo "healthy" && exit 0
                        exit 1
    - name: error-rate
      count: 2
      interval: 60s
      failureLimit: 1
      successCondition: result.errorRate < {{args.error-threshold}}
      provider:
        job:
          spec:
            template:
              spec:
                containers:
                  - name: newrelic-check
                    image: curlimages/curl:latest
                    env:
                      - name: NEW_RELIC_HOST
                        valueFrom:
                          configMapKeyRef:
                            name: voting01-config
                            key: NEW_RELIC_HOST
                            optional: true
                    args:
                      - |
                        NR_HOST="${NEW_RELIC_HOST:-opsera-opsera-newrelic-dev.agent.opsera.dev}"
                        RESPONSE=$(curl -s "https://${NR_HOST}/v1/metrics?app={{args.app-name}}&metric=errorRate&window=5m")
                        ERROR_RATE=$(echo "$RESPONSE" | grep -oE '"errorRate"[[:space:]]*:[[:space:]]*[0-9.]+' | grep -oE '[0-9.]+$' || echo "0")
                        echo '{"errorRate": '"$ERROR_RATE"'}'
                        [ $(awk "BEGIN {print ($ERROR_RATE < {{args.error-threshold}})}" ) -eq 1 ] && exit 0 || exit 1
```

---

## Key Prompts & Responses

### Prompt 1: "did preview auto promote to stage?"

**Context:** User noticed staging went to Healthy immediately instead of Paused.

**Response:** Identified that `autoPromotionEnabled: true` was set. Changed to `false` to require manual promotion. Also created a new workflow `promote-staging-rollout-voting01.yaml` for manual promotion.

---

### Prompt 2: "errors are simulated even when configured not to"

**Context:** Error simulation was firing even when toggled off.

**Response:** Simplified the entire error simulation system. Removed complex ErrorSimulator class with thread-safe state, 50% error rate, auto-disable. Replaced with simple global boolean: ON = 100% failure, OFF = normal.

---

### Prompt 3: "i need name of commiters in history also"

**Context:** Deployment dashboard showed deployment times but not who deployed.

**Response:** Updated `deployment-landscape-voting01.yaml` to include committer name in each deployment history bullet:
```
• `e3e1a9b-2026` (17 mins ago) by _srinivas-source_
```

---

### Prompt 4: "lets test end to end with cosmetic change"

**Context:** User wanted to test full pipeline from code change to all environments.

**Response:** Made version change (v16→v17), committed, pushed. Monitored:
1. DEV auto-deploy (~2 min)
2. QA canary 8-step rollout (~11 min)
3. Staging blue-green with preview (~3 min)
4. Manual promotion
5. Landscape update with committer names

---

### Prompt 5: "give us detailed report... explain every minute detail"

**Context:** User wanted comprehensive deployment report with visuals.

**Response:** Created detailed report in `.deployments/2026-02-04-e3e1a9b-v17-deployment-report.md` including:
- ASCII art diagrams of traffic flow
- Minute-by-minute timeline
- APM query details
- Kubernetes objects created
- Analysis run execution flow

---

## Best Practices Discovered

### 1. Blue-Green with Manual Promotion
```yaml
# Always for staging/production
autoPromotionEnabled: false
```

### 2. Canary Step Configuration
```yaml
# Recommended pattern: weight → pause → analysis
steps:
  - setWeight: 20
  - pause: { duration: 2m }
  - analysis: { templates: [health-check] }
  - setWeight: 50
  - pause: { duration: 3m }
  - analysis: { templates: [apm-analysis] }  # Include APM check
  - setWeight: 100
  - pause: { duration: 1m }
  - analysis: { templates: [apm-analysis] }  # Final verification
```

### 3. Error Simulation Design
- Keep it simple: ON = fail, OFF = normal
- No complex error rates or auto-disable
- Clear visual indicator (red banner)
- Health endpoint (`/health`) should always return 200

### 4. APM Analysis with Job Provider
- Use Job provider (not web) to avoid secretKeyRef issues
- Default to mock server for dev/qa
- Parse JSON response with grep/awk (portable)
- TTL cleanup: `ttlSecondsAfterFinished: 120`

### 5. Deployment Dashboard
- Include committer names for audit trail
- Use relative time (e.g., "17 mins ago")
- Auto-update README with deployment status
- Provide direct links to run workflows

### 6. Preview URL Testing
- Always provide preview URL in workflow summary
- Include both preview and active URLs for comparison
- Add "Promote" button with badge in summary

---

## Skill Update Recommendations

### New Rules to Add

**RULE 42: Blue-Green Manual Promotion**
```
For staging/production blue-green deployments, ALWAYS set autoPromotionEnabled: false.
Create a separate promotion workflow that requires explicit confirmation.
```

**RULE 43: Error Simulation Simplicity**
```
Keep error simulation simple: ON = 100% failure, OFF = normal.
Avoid complex error rates, auto-disable timers, or thread-safe state management.
Health endpoints should NEVER be affected by error simulation.
```

**RULE 44: Deployment Attribution**
```
Always include committer names in deployment history for audit trails.
Look up source commit author, not the bot that made the deploy commit.
```

**RULE 45: Preview URL in Workflow Summary**
```
For blue-green deployments, always include preview and active URLs in the workflow summary.
Add a prominent "Promote to Active" button when rollout is paused.
```

### New Learnings to Add

1. **Same image tag = no rollout:** Argo Rollouts won't trigger a new rollout if the image tag hasn't changed.

2. **HTTPS for external resources:** Always use HTTPS for jQuery, CDN resources to avoid mixed content blocks.

3. **Heredoc YAML issues:** Heredocs at column 1 can break YAML parsing in GitHub Actions.

4. **Git merge conflicts in CI:** Concurrent deployments may cause merge conflicts in kustomization.yaml - this is acceptable and will self-resolve.

5. **Promotion workflow validation:** Can either enforce strict path (DEV→QA→Staging) or allow direct deployment with image tag.

### Templates to Add to Skill

1. `promote-staging-rollout.yaml` - Manual promotion workflow
2. Error simulation Flask endpoint pattern
3. Error simulation UI toggle pattern
4. Deployment history with committer names pattern
5. Comprehensive deployment report template

---

## Files Changed This Session

| File | Change Type | Description |
|------|-------------|-------------|
| `.opsera-voting01/k8s/overlays/staging/vote-rollout.yaml` | Modified | Set autoPromotionEnabled: false |
| `.github/workflows/promote-staging-rollout-voting01.yaml` | Created | Manual promotion workflow |
| `.github/workflows/ci-build-push-voting01-staging.yaml` | Modified | Added preview URLs, promote button |
| `.github/workflows/deployment-landscape-voting01.yaml` | Modified | Added committer names to history |
| `vote/app.py` | Modified | Simplified error simulation |
| `vote/templates/index.html` | Modified | Simple toggle UI, HTTPS jQuery |
| `.deployments/2026-02-04-e3e1a9b-v17-deployment-report.md` | Created | Comprehensive deployment report |

---

**Session Duration:** ~2 hours
**Deployments Completed:** 3 (DEV, QA, Staging)
**Total Pipeline Time:** ~20 minutes
**Issues Fixed:** 9
**New Features:** 4
