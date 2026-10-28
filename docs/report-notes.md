# Notes for the report

- Business requirements: visibility into utilization, health, user activity and security configuration; alerts on
  abnormal conditions; an audit trail; protection against unnecessary exposure.
- Design decisions: CloudWatch plus SNS for monitoring and alerts; a multi-region CloudTrail trail to a private S3 bucket
  for auditing; EventBridge for near real-time security-change alerts; hardening limited to measures that do not break
  existing work.
- Security improvements: SSH narrowed from the whole internet to one address, IMDSv2 required, new disks encrypted,
  stronger password policy, log bucket private with TLS-only access and log file validation.
- Operational value: the dashboard and alarms show health at a glance; CloudTrail answers who changed what and when;
  alerts shorten the time to notice a problem.
- Limitations: five-minute metric granularity, no in-server logs or application metrics, a single notification channel,
  AdministratorAccess on the working user (least privilege would be the next improvement).
- Cost: small, because the services used are free or very cheap at this scale; the main cost risk is leaving servers running.
