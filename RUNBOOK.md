# Experiment 7 runbook (Windows cmd, region ap-south-1)

Run commands ONE AT A TIME and keep the double quotes. Each command must stay on one line.
Screenshot labels are S1, S2, and so on. Never screenshot a secret value.
Do the hardening phase (7) LAST, because it changes who can log in over SSH.

## Phase 0 - Reuse earlier setup, then publish the repo

Access keys, the AWS CLI, the GitHub CLI login and your Git name and email from the earlier experiments are reused.
Nothing needs to be re-created, and no keys go into this repository.

    aws sts get-caller-identity
    gh auth status
    git --version

Unpack and publish:

    cd %USERPROFILE%
    tar -xf %USERPROFILE%\Downloads\exp7-cloud-monitoring-audit.zip
    cd exp7-cloud-monitoring-audit
    git init -b main
    git add .
    git commit -m "Experiment 7: monitoring, auditing and hardening runbook and configs"
    gh repo create Satwik7K/exp7-cloud-monitoring-audit --public --source=. --remote=origin --push

(If `git init -b` is not recognised: `git init`, then `git branch -M main`.)

Shortcuts (re-run these in any new cmd window):

    for /f %i in ('aws sts get-caller-identity --query Account --output text') do set ACCOUNT=%i
    for /f %i in ('aws ec2 describe-instances --filters "Name=tag:Name,Values=cicd-server" "Name=instance-state-name,Values=running,stopped" --query "Reservations[0].Instances[0].InstanceId" --output text') do set INSTANCE=%i
    echo %ACCOUNT% %INSTANCE%

If INSTANCE says None, tell me. If the server is stopped, start it (alarms need live metrics):

    aws ec2 start-instances --instance-ids %INSTANCE%
    aws ec2 wait instance-running --instance-ids %INSTANCE%

## Phase 1 - Baseline: the "before" picture (S1)

Why: shows the visibility and exposure problem before we fix it.

    aws cloudtrail describe-trails --query "trailList[].Name" --output text
    aws cloudwatch describe-alarms --query "MetricAlarms[].AlarmName" --output text
    aws ec2 describe-security-groups --group-names cicd-sg --query "SecurityGroups[0].IpPermissions[].[FromPort,IpRanges[0].CidrIp]" --output table
    aws ec2 describe-instances --instance-ids %INSTANCE% --query "Reservations[0].Instances[0].MetadataOptions.HttpTokens"
    aws ec2 get-ebs-encryption-by-default
    aws iam get-account-password-policy

Expect: no trail, no alarms, SSH (22) open to 0.0.0.0/0, tokens "optional", encryption false, and a NoSuchEntity
error for the password policy (that error is the "before" evidence). S1 = these outputs.

## Phase 2 - Alert channel: SNS email (S2)

    aws sns create-topic --name exp7-alerts
    set TOPIC=arn:aws:sns:ap-south-1:%ACCOUNT%:exp7-alerts
    aws sns subscribe --topic-arn %TOPIC% --protocol email --notification-endpoint YOUR_EMAIL@example.com

Open the "Subscription Confirmation" email (check spam) and click Confirm subscription. Then:

    aws sns list-subscriptions-by-topic --topic-arn %TOPIC% --query "Subscriptions[*].[Protocol,Endpoint,SubscriptionArn]" --output table

S2 = this table showing a full ARN (not PendingConfirmation).

## Phase 3 - Monitoring: alarms and dashboard (S3, S4)

    aws cloudwatch put-metric-alarm --alarm-name exp7-cpu-high --alarm-description "CPU above 70 percent" --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=%INSTANCE% --statistic Average --period 300 --evaluation-periods 1 --threshold 70 --comparison-operator GreaterThanThreshold --alarm-actions %TOPIC% --ok-actions %TOPIC%
    aws cloudwatch put-metric-alarm --alarm-name exp7-status-check-failed --alarm-description "EC2 status check failed" --namespace AWS/EC2 --metric-name StatusCheckFailed --dimensions Name=InstanceId,Value=%INSTANCE% --statistic Maximum --period 300 --evaluation-periods 1 --threshold 0 --comparison-operator GreaterThanThreshold --treat-missing-data notBreaching --alarm-actions %TOPIC%
    aws cloudwatch describe-alarms --alarm-names exp7-cpu-high exp7-status-check-failed --query "MetricAlarms[*].[AlarmName,StateValue,Threshold]" --output table

New alarms show INSUFFICIENT_DATA for a few minutes, then OK. Re-run the last command until both say OK.
Resource utilization readings (widen the dates if empty):

    aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=%INSTANCE% --start-time 2026-10-27T00:00:00Z --end-time 2026-10-30T00:00:00Z --period 300 --statistics Average --query "sort_by(Datapoints,&Timestamp)[-6:].[Timestamp,Average]" --output table

S3 = the alarm table (OK) and the CPU readings.

Dashboard (the template placeholders are filled in by PowerShell):

    powershell -Command "(Get-Content config\dashboard.json) -replace 'INSTANCE_ID','%INSTANCE%' -replace 'ACCOUNT_ID','%ACCOUNT%' | Set-Content config\dashboard.live.json"
    aws cloudwatch put-dashboard --dashboard-name exp7-dashboard --dashboard-body file://config/dashboard.live.json
    aws cloudwatch list-dashboards --query "DashboardEntries[].DashboardName"

S4 = browser screenshot: AWS Console, CloudWatch, Dashboards, exp7-dashboard (Mumbai region).

## Phase 4 - Test the alerts (S5, S6)

Instant test of the whole path (alarm, SNS, email):

    aws cloudwatch set-alarm-state --alarm-name exp7-cpu-high --state-value ALARM --state-reason "Manual test for Experiment 7"
    aws cloudwatch describe-alarm-history --alarm-name exp7-cpu-high --max-items 5 --query "AlarmHistoryItems[*].[Timestamp,HistorySummary]" --output table

S5 = the email titled ALARM: "exp7-cpu-high". S6 = the alarm history table.

Optional, a real CPU alert (takes 6 to 10 minutes; costs pennies at most). Run this BEFORE Phase 7:

    for /f %i in ('aws ec2 describe-instances --instance-ids %INSTANCE% --query "Reservations[0].Instances[0].PublicIpAddress" --output text') do set IP=%i
    ssh -i %USERPROFILE%\.ssh\cicd-key.pem ec2-user@%IP% "for i in 1 2; do nohup timeout 480 sh -c 'while true; do :; done' >/dev/null 2>&1 & done"

The load stops by itself after 8 minutes. Watch with the describe-alarms command until the state says ALARM.

## Phase 5 - Auditing: CloudTrail (S7, S8, S9)

Why: CloudTrail records every API call, so you can answer who did what and when.

    set BUCKET=exp7-trail-%ACCOUNT%
    aws s3api create-bucket --bucket %BUCKET% --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1
    aws s3api put-public-access-block --bucket %BUCKET% --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
    powershell -Command "(Get-Content config\cloudtrail-bucket-policy.json) -replace 'BUCKET_NAME','%BUCKET%' -replace 'ACCOUNT_ID','%ACCOUNT%' | Set-Content config\cloudtrail-bucket-policy.live.json"
    aws s3api put-bucket-policy --bucket %BUCKET% --policy file://config/cloudtrail-bucket-policy.live.json
    aws cloudtrail create-trail --name exp7-trail --s3-bucket-name %BUCKET% --is-multi-region-trail --enable-log-file-validation
    aws cloudtrail start-logging --name exp7-trail
    aws cloudtrail get-trail-status --name exp7-trail --query "[IsLogging,LatestDeliveryTime]" --output table

S7 = IsLogging true. Audit evidence (earlier activity is visible too, because event history keeps 90 days):

    aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateTrail --max-results 3 --query "Events[].[EventTime,EventName,Username]" --output table
    aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances --max-results 3 --query "Events[].[EventTime,EventName,Username]" --output table

S8 = these tables (user, event, time). Wait 5 to 15 minutes, then:

    aws s3 ls s3://%BUCKET%/AWSLogs/ --recursive

S9 = log files in the bucket. If nothing is listed yet, wait and repeat.

## Phase 6 - Security alert: firewall changes (S10, S11, S12)

Why: a changed firewall rule can expose a server, so we want an email the moment it happens.

    aws events put-rule --name exp7-sg-change --event-pattern file://config/events-pattern.json --description "Alert when a security group rule is changed"
    powershell -Command "(Get-Content config\sns-topic-policy.json) -replace 'TOPIC_ARN','%TOPIC%' -replace 'ACCOUNT_ID','%ACCOUNT%' | Set-Content config\sns-topic-policy.live.json"
    aws sns set-topic-attributes --topic-arn %TOPIC% --attribute-name Policy --attribute-value file://config/sns-topic-policy.live.json
    powershell -Command "(Get-Content config\events-targets.json) -replace 'TOPIC_ARN','%TOPIC%' | Set-Content config\events-targets.live.json"
    aws events put-targets --rule exp7-sg-change --targets file://config/events-targets.live.json
    aws events describe-rule --name exp7-sg-change --query "[Name,State]"
    aws events list-targets-by-rule --rule exp7-sg-change --query "Targets[].[Id,Arn]"

S10 = rule ENABLED and its SNS target. Trigger it with a throwaway security group:

    aws ec2 create-security-group --group-name exp7-test-sg --description "Experiment 7 test"
    aws ec2 authorize-security-group-ingress --group-name exp7-test-sg --protocol tcp --port 8080 --cidr 10.0.0.0/8

Within a minute or two you should get an email about AuthorizeSecurityGroupIngress. S11 = that email. Then the audit record
(may take a few minutes to appear), and clean up the test group:

    aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress --max-results 3 --query "Events[].[EventTime,EventName,Username]" --output table
    aws ec2 delete-security-group --group-name exp7-test-sg

S12 = the lookup table showing your user making the change.

## Phase 7 - Security hardening, LAST (S13)

Why: each command closes one exposure.

1. SSH only from your own IP (was open to the whole internet). NOTE: this stops the Experiment 4 GitHub Actions
   deploy from reaching the server. To undo: revoke your /32 rule and add 0.0.0.0/0 again.

       for /f %i in ('curl -s https://checkip.amazonaws.com') do set MYIP=%i
       echo %MYIP%
       aws ec2 revoke-security-group-ingress --group-name cicd-sg --protocol tcp --port 22 --cidr 0.0.0.0/0
       aws ec2 authorize-security-group-ingress --group-name cicd-sg --protocol tcp --port 22 --cidr %MYIP%/32

2. Require IMDSv2 on the server (blocks a common credential-theft technique):

       aws ec2 modify-instance-metadata-options --instance-id %INSTANCE% --http-tokens required --http-endpoint enabled

3. Encrypt new disks by default:

       aws ec2 enable-ebs-encryption-by-default

4. Strong IAM password policy:

       aws iam update-account-password-policy --minimum-password-length 12 --require-symbols --require-numbers --require-uppercase-characters --require-lowercase-characters --max-password-age 90 --password-reuse-prevention 5

5. MFA status (the root account should show 1):

       aws iam get-account-summary --query "SummaryMap.[AccountMFAEnabled,Users]"

The log bucket was already hardened in Phase 5 (public access blocked, TLS-only policy). We deliberately do NOT turn on the
account-wide S3 public access block: if any of your buckets serves a public website, it would take that site offline.

Verify the "after" state:

    aws ec2 describe-security-groups --group-names cicd-sg --query "SecurityGroups[0].IpPermissions[].[FromPort,IpRanges[0].CidrIp]" --output table
    aws ec2 describe-instances --instance-ids %INSTANCE% --query "Reservations[0].Instances[0].MetadataOptions.HttpTokens"
    aws ec2 get-ebs-encryption-by-default
    aws iam get-account-password-policy --query "PasswordPolicy.[MinimumPasswordLength,RequireSymbols,MaxPasswordAge]"
    aws s3api get-public-access-block --bucket %BUCKET%

S13 = these outputs next to S1.

## Test cases (fill the Actual Result column from your screenshots)

See the table in README.md (T1 to T10): T1 S2, T2 S3, T3 S5 and S6, T4 S4, T5 S7 and S9, T6 S8, T7 S10 to S12,
T8 S13, T9 S13, T10 S13.

## Screenshot checklist
S1 baseline | S2 SNS subscription | S3 alarms OK + CPU readings | S4 dashboard (browser) | S5 alert email | S6 alarm history |
S7 trail logging | S8 audit lookup | S9 log files in S3 | S10 EventBridge rule + target | S11 security email |
S12 audit of the change | S13 hardening after | S14 GitHub repo page

## Cleanup (after the report is done)

    aws cloudwatch delete-alarms --alarm-names exp7-cpu-high exp7-status-check-failed
    aws cloudwatch delete-dashboards --dashboard-names exp7-dashboard
    aws events remove-targets --rule exp7-sg-change --ids sns-alert
    aws events delete-rule --name exp7-sg-change
    aws cloudtrail delete-trail --name exp7-trail
    aws s3 rm s3://%BUCKET% --recursive
    aws s3api delete-bucket --bucket %BUCKET%
    aws sns delete-topic --topic-arn %TOPIC%
    aws ec2 stop-instances --instance-ids %INSTANCE%

The hardening changes are good practice and can stay.
