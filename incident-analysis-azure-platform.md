# Incident Analysis – Azure Platform Failure

## Overview

A deployment was triggered through GitHub Actions, which ran an OpenTofu apply. This deployment updated three critical components:

- Network Security Group (NSG) rules  
- Key Vault access policy  
- TLS certificate  

The deployment completed successfully with no errors. However, around 20 minutes later, multiple alerts were triggered in Datadog, indicating a system-wide issue.

---

## Problem Observed

After the deployment, the following issues were noticed:

- AKS pods started crash-looping  
- Users received 502 errors from Application Gateway  
- Downstream API calls started timing out  

This clearly indicated that the application was no longer functioning correctly across multiple layers.

---

## Root Cause

The root cause was a faulty deployment that introduced multiple misconfigurations at the same time across:

- Key Vault access policy  
- NSG outbound rules  
- TLS certificate configuration  

These issues were interconnected and led to a cascading failure across the system.

---

## Task 1 — Root Cause Analysis

To understand the issue, I followed a structured debugging approach.

First, I checked what changed during the deployment. The NSG rules, Key Vault policy, and TLS certificate were all modified, which gave an initial direction.

Then I analyzed the system layer by layer:

- **AKS Pods**  
  The pod logs showed errors like "access denied while retrieving secrets."  
  This confirmed that the application lost access to Key Vault, causing pods to crash repeatedly.

- **Application Gateway**  
  The gateway was returning 502 errors.  
  This was happening because:
  - Pods were not healthy (no backend available), and  
  - There was also a TLS mismatch due to certificate changes.

- **NSG Rules**  
  On reviewing outbound rules, it was clear that traffic to the downstream API was blocked.  
  This caused API requests to timeout.

All these issues combined resulted in a cascading failure.

---

## Task 2 — Fix Strategy

Instead of doing a full rollback, I evaluated the risks.

A rollback was not ideal because the last stable deployment was 3 weeks old, which could break other working changes.

So I decided to fix each issue individually:

- **Key Vault Issue**  
  Restored the correct access policy.  
  This immediately stopped the crash-looping of pods.

- **NSG Issue**  
  Fixed the outbound rule to allow communication with the downstream API.  
  This resolved the timeout issue.

- **TLS Issue**  
  Updated the Application Gateway to correctly use the new certificate.  
  This fixed the 502 errors.

The priority was to restore application availability as quickly as possible.

---

## Task 3 — Observability using Datadog MoC

Datadog helped confirm both the issue and the root cause.

- **Metrics** showed:
  - Increase in pod restarts  
  - Spike in 5xx errors  
  - Increase in latency  

- **Logs** revealed:
  - "Access denied" errors → Key Vault issue  
  - TLS handshake failures → certificate issue  
  - Timeout errors → network issue  

- **Traces (APM)** helped track where requests were failing:
  - At startup (Key Vault issue)  
  - During API calls (NSG issue)  

All signals aligned with the deployment timing, confirming that the deployment triggered the failure.

---

## Task 4 — Prevention

### Why This Happened

- Multiple critical changes were deployed together  
- No proper validation before deployment  
- No safe rollout strategy  

---

### Improvements

#### CI/CD Enhancements

- Add pre-deployment validation checks  
- Introduce staging environment for testing  
- Add approval gates for critical changes  

---

#### Deployment Strategy

- Use canary deployments for gradual rollout  
- Use blue-green deployments for safer switching  
- Avoid bundling multiple risky changes together  

---

#### Infrastructure Safeguards

- Enable state file versioning  
- Maintain version-controlled configurations  
- Add policy validation to prevent bad configs  

---

#### Observability Improvements

- Create dashboards for system health  
- Add proactive alerts for:
  - Key Vault failures  
  - TLS issues  
  - Network failures  

---

## ✅ Final Summary

A single deployment introduced multiple configuration issues across Key Vault, NSG, and TLS.  

- Key Vault issue caused pods to crash  
- NSG issue caused API timeouts  
- TLS issue caused 502 errors  

These combined into a cascading failure.

The issue was resolved by fixing each component individually, and future prevention focuses on better validation, safer deployments, and improved monitoring.