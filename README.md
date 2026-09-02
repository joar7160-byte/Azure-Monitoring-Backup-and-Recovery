# Monitoring, Backup, and Recovery Lab

## Overview

A business needs to know when its infrastructure is under stress, recover data after accidental loss, and prove that a disaster recovery plan actually works, not just that it exists on paper. This project builds that environment in Azure: a monitored virtual machine with alerting on high CPU, a Recovery Services vault backing it up on a daily schedule, an attempted cross-region disaster recovery configuration, and Log Analytics queries used to investigate performance data directly.

**Scope:** 1 virtual machine, VM Insights and a CPU alert rule verified to actually fire, 1 Recovery Services vault with a daily backup policy and a completed recovery point, 1 Site Recovery configuration attempt, 1 Log Analytics workspace queried with KQL, and vault encryption confirmed.

## Skills & Tools

| Category | Tools/Skills |
|---|---|
| Cloud Infrastructure | Microsoft Azure, Resource Groups, Virtual Machines, Managed Identities |
| Monitoring | Azure Monitor, VM Insights, OpenTelemetry metrics, metric alert rules, action groups |
| Backup | Azure Backup, Recovery Services Vault, backup policies, on-demand backup, encryption |
| Disaster Recovery | Azure Site Recovery, cross-region replication, Automation Accounts |
| Log Analytics | Data Collection Rules, Azure Monitor Agent, KQL |
| Troubleshooting | Azure Policy allowed locations, deployment validation errors, agent data pipelines |

## Environment

- **Resource Group:** `RG-Monitoring`
- **Virtual Machine:** `test-vm`, Windows Server 2022 Datacenter: Azure Edition, Trusted Launch enabled
- **Region:** East US 2
- **Recovery Services Vault:** `rsv-monitoring`, Locally-redundant storage
- **Log Analytics Workspace:** `law-monitoring`
- **Data Collection Rule:** `dcr-perf-to-law`, Performance Counters to Log Analytics
- **Alert Rule:** Percentage CPU greater than 80%, email action group

## Steps

**1. Created the virtual machine**

<img src="screenshots/01-vm-create-review.png" width="700"><br><br>

**2. Enabled VM Insights and configured detailed monitoring**

Enabled OpenTelemetry metrics and recommended alert rules through the VM's Insights blade. This required creating a user-assigned managed identity, since the wizard only supported that identity type in this scenario, not system-assigned.

<img src="screenshots/02-vm-insights-before-data.png" width="700"><br><br>
<img src="screenshots/03-configure-monitor-capabilities.png" width="700"><br><br>
<img src="screenshots/04-configure-monitor-review.png" width="700"><br><br>

**3. Configured and verified the CPU alert rule**

Set the Percentage CPU alert threshold to 80%, with an email action group attached.

<img src="screenshots/05-alert-rule-cpu-80-percent.png" width="500"><br><br>
<img src="screenshots/06-alert-rules-list.png" width="900"><br><br>
<img src="screenshots/07-alert-rule-detail-scope-actions.png" width="700"><br><br>

Stress-tested the VM's CPU past 80% using a PowerShell loop and checked the rule's evaluation history, which initially showed no evaluations at all despite real, sustained CPU load confirmed in Task Manager.

<img src="screenshots/08-troubleshooting-alert-history-empty.png" width="700"><br><br>

The alert eventually fired and delivered an email notification, confirming the rule was correctly configured and the delay was in evaluation timing, not a broken pipeline.

<img src="screenshots/09-alert-fired-email.png" width="500"><br><br>

**4. Set up Azure Backup**

Created a Recovery Services vault and disabled Geo-redundant storage in favor of Locally-redundant storage, since cross-region resilience is demonstrated separately through Site Recovery.

<img src="screenshots/10-recovery-services-vault-overview.png" width="700"><br><br>

The Standard backup policy sub-type was rejected because the VM uses Trusted Launch, which requires the Enhanced policy sub-type instead.

<img src="screenshots/11-troubleshooting-standard-policy-trusted-launch-error.png" width="900"><br><br>

Switched to Enhanced, created a daily policy with 7-day retention, and enabled backup.

<img src="screenshots/12-configure-backup-enhanced-policy.png" width="700"><br><br>
<img src="screenshots/13-backup-policy-daily-7day-retention.png" width="500"><br><br>
<img src="screenshots/14-backup-jobs-configure-completed.png" width="900"><br><br>

Triggered a manual on-demand backup and verified it completed with a real recovery point.

<img src="screenshots/15-backup-item-initial-pending.png" width="900"><br><br>
<img src="screenshots/16-backup-item-recovery-point-success.png" width="900"><br><br>

**5. Attempted Site Recovery replication to a secondary region**

<img src="screenshots/17-site-recovery-basics-target-region.png" width="700"><br><br>

Deployment failed due to a subscription-level policy violation when Site Recovery tried to auto-create an Automation Account in the target region.

<img src="screenshots/18-troubleshooting-asr-deployment-validation-failed.png" width="700"><br><br>

Switched target regions and pulled the subscription's actual allowed-locations policy directly, which listed `mexicocentral, southcentralus, northcentralus, eastus2, canadacentral`. A separate Free Trial and Student subscription restriction limits Automation Account creation to `eastus, eastus2, westus, northeurope, southeastasia, japanwest`. The only region satisfying both lists is `eastus2`, the source region itself, making cross-region replication infeasible on this subscription tier.

<img src="screenshots/19-site-recovery-target-region-central-us.png" width="700"><br><br>
<img src="screenshots/20-troubleshooting-asr-automation-account-region-blocked.png" width="700"><br><br>

**6. Queried performance data in Log Analytics**

No Log Analytics workspace existed yet, since the earlier monitoring setup routed data to an Azure Monitor workspace instead.

<img src="screenshots/21-troubleshooting-no-log-analytics-workspace.png" width="900"><br><br>

Created a workspace and a Data Collection Rule to route performance counters to it, confirmed via its diagram that both the OpenTelemetry pipeline and the new Log Analytics pipeline were correctly configured side by side.

<img src="screenshots/22-dcr-correctly-configured-diagram.png" width="700"><br><br>

Queried CPU performance data from the stress test, confirming real values averaging 100% across the sampled window.

<img src="screenshots/23-kql-query-success-avg-cpu-100.png" width="700"><br><br>

**7. Verified backup encryption**

Confirmed the Recovery Services vault encrypts data at rest using Microsoft-managed keys by default.

<img src="screenshots/24-vault-encryption-microsoft-managed-keys.png" width="700"><br><br>

## Decisions & Significance

- **Locally-redundant storage for the vault instead of Geo-redundant.** Cross-region resilience is what Site Recovery is for. Paying for Geo-redundant backup storage on top of that would have been redundant cost for a requirement already covered elsewhere in the same project.

- **Diagnosed the alert evaluation delay with evidence instead of assuming misconfiguration.** The alert rule's empty evaluation history was checked directly rather than guessed at, which is what confirmed the real cause was a delay in evaluation rather than a broken rule.

- **Documented the Site Recovery limitation with the actual policy data.** Rather than retrying regions indefinitely, the subscription's exact allowed-locations list was pulled directly from Azure Policy and cross-referenced against the Automation Account's own regional restriction, confirming the two lists only overlap on the source region itself.

## Troubleshooting Notes

- Standard backup policy sub-type was rejected for a Trusted Launch VM; resolved by switching to Enhanced, which explicitly supports Trusted Launch
- Site Recovery deployment failed with a policy violation when auto-creating an Automation Account in the target region; confirmed via Activity Log error details and the subscription's actual allowed-locations policy that no region satisfies both the general location policy and the Student subscription's Automation Account restriction other than the source region
- The CPU alert rule's evaluation history showed no results despite sustained real CPU load; the alert eventually fired and delivered an email, confirming a delay in evaluation rather than a configuration problem

## What This Demonstrates

- Configuring Azure Monitor, VM Insights, and a metric alert rule, and verifying the alert actually fires rather than only appearing correctly configured
- Setting up Azure Backup with a policy matched to the VM's actual security configuration, and completing a real, verified recovery point
- Diagnosing a genuine subscription-level infrastructure constraint using policy data rather than trial and error, and documenting it as a real finding
- Writing and running a KQL query against live performance data collected through a Data Collection Rule

## Why This Project Matters

Monitoring and backup are only useful if they actually work when needed, not just when they look correctly configured. This project shows that distinction throughout: an alert that appeared silent until its evaluation history and eventual firing were checked directly, a backup policy that had to match the VM's real security settings before it would even save, and a disaster recovery attempt that hit a genuine platform constraint, diagnosed with the actual policy data rather than guessed at. That's the same kind of verification a real environment demands before anyone can trust that monitoring or recovery will hold up during an actual incident.
