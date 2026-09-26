# Incident: Site unreachable after probe port misconfiguration

## Summary
A change to the ARM template's `probePort` parameter caused the Azure Load
Balancer's health probe to check port 8080 instead of 80, where the backend
nginx servers actually listen. All backends were marked unhealthy and the
load balancer stopped routing traffic, taking the site offline.

## Impact
- Site fully unreachable (connection timeouts) for all users.
- Duration: ~21 minutes (05:09–05:30 UTC+1)

## Timeline (UTC+1)
- 05:09:19 — "Change probe port" commit (9c30c6a) pushed to main; pipeline validated and deployed successfully
- 05:09:19–05:30:00 — Health Probe Status dropped to 0%, load balancer stopped routing traffic; Invoke-WebRequest confirmed timeout
- 05:30:00 — "Revert" commit (f436d23) pushed to main; pipeline redeployed the corrected probePort value
- shortly after 05:30:00 — Health Probe Status recovered to 100%, site returning 200 OK

## Detection
Detected via the Azure Monitor metric alert on Health Probe Status
(DipAvailability), backed by direct confirmation via Invoke-WebRequest.

## Root Cause
The ARM template's `probePort` parameter was changed from 80 to 8080.
The load balancer's health probe began checking port 8080, but nginx on
both backend VMs only listens on port 80. The probe failed for every
backend, so the load balancer marked all of them unhealthy and stopped
forwarding traffic — even though the VMs and application were completely
healthy throughout.

## Resolution
Verified nginx was healthy on the VMs (systemctl status, ss -tlnp) to rule
out an application-level failure. Inspected the load balancer's probe
config directly (az network lb probe show) and confirmed the port
mismatch. Reverted the offending commit and pushed, which redeployed the
correct probePort value of 80 through the CI/CD pipeline.

## What went well
- The pipeline deployed the broken config successfully but the monitoring
  alert caught the resulting outage within minutes.
- Having IaC in git meant the fix was a one-line revert, not manual portal
  changes.

## What didn't go well
- The CI/CD pipeline's `validate` and `what-if` steps only check that the
  ARM template is syntactically valid and deployable — they don't catch
  application-level misconfigurations like a probe pointing at the wrong
  port.

## Prevention
- Add a post-deploy smoke test step to the pipeline that curls the load
  balancer's public IP and fails the job if it doesn't get a 200.
- Consider a manual approval gate on production deploys after `what-if`.
- Alert on Data Path Availability in addition to Health Probe Status for
  broader coverage.