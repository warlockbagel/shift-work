# EC2 Serial Console Event-Driven Security Monitoring

This CloudFormation template provides **real-time, event-driven monitoring** for EC2 Serial Console access with immediate automated response capabilities. Unlike polling-based solutions, this template responds within **~5 minutes average (1-15 minutes range)** of any `EnableSerialConsoleAccess` event.

## 🚀 Overview

This solution monitors for EC2 Serial Console enablement events via CloudTrail and responds immediately with:
- **Real-time detection** via EventBridge rules
- **Automatic remediation** via SSM automation (optional)
- **Instant notifications** to security teams
- **Comprehensive logging** and troubleshooting capabilities

### Architecture

```
CloudTrail Event → EventBridge Rule → SSM Automation → [Disable + Notify]
   (~5 min avg)        (immediate)        (immediate)      (immediate)
```

## 📋 Prerequisites

### Required AWS Services
1. **AWS CloudTrail** - Multi-region trail with management events enabled
2. **Amazon EventBridge** - Default event bus (automatically available)
3. **AWS Systems Manager** - For automation execution
4. **Amazon SNS** - For notifications

### Permissions Required
- CloudFormation deployment permissions
- IAM role creation permissions  
- EventBridge, SSM, SNS, and EC2 permissions

### CloudTrail Configuration
**Critical**: You must have a CloudTrail trail configured to capture management events:

```bash
# Check if CloudTrail is enabled
aws cloudtrail describe-trails

# Create a trail if needed (recommended: multi-region)
aws cloudtrail create-trail \
    --name ec2-security-monitoring \
    --s3-bucket-name your-cloudtrail-bucket \
    --include-global-service-events \
    --is-multi-region-trail

# Start logging
aws cloudtrail start-logging --name ec2-security-monitoring
```

## 🛠️ Deployment

### Quick Deploy
```bash
aws cloudformation create-stack \
  --stack-name ec2-serial-console-event-monitoring \
  --template-body file://ec2-serial-console-monitoring-event-driven.yaml \
  --parameters ParameterKey=NotificationEmail,ParameterValue=security@yourcompany.com \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Custom Configuration
```bash
aws cloudformation create-stack \
  --stack-name ec2-serial-console-event-monitoring \
  --template-body file://ec2-serial-console-monitoring-event-driven.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=security@yourcompany.com \
    ParameterKey=AutoRemediation,ParameterValue=false \
  --capabilities CAPABILITY_NAMED_IAM
```

### Parameters

| Parameter | Description | Default | Options |
|-----------|-------------|---------|---------|
| `NotificationEmail` | Email address for security alerts | *Required* | Valid email address |
| `AutoRemediation` | Automatically disable serial console when enabled | `true` | `true`, `false` |

## 🔧 How It Works

### Event Flow
1. **Event Trigger**: Someone calls `aws ec2 enable-serial-console-access` or uses AWS Console
2. **CloudTrail Capture**: CloudTrail logs the `EnableSerialConsoleAccess` API call
3. **EventBridge Detection**: EventBridge rule matches the event pattern (~5 minutes average, 1-15 min range)
4. **SSM Automation**: EventBridge triggers SSM automation document immediately
5. **Status Check**: SSM checks current serial console access status
6. **Remediation Logic**:
   - If `AutoRemediation=true`: Automatically disables serial console access
   - If `AutoRemediation=false`: Sends alert notification only
7. **Notification**: SNS sends detailed security alert regardless of remediation action

### Event Pattern Detection
The EventBridge rule detects:
- **API Calls**: Direct calls to `EnableSerialConsoleAccess` via CLI/SDK
- **Console Actions**: AWS Management Console actions (if they trigger API calls)
- **Cross-Account**: Events from assumed roles (if CloudTrail captures them)

## 📊 Resources Created

### Core Components
- **EventBridge Rule**: `EC2SerialConsoleEnableDetection`
- **SSM Automation Document**: `EC2SerialConsoleImmediateRemediation`
- **SNS Topic**: `EC2SerialConsoleSecurityAlerts`
- **IAM Roles**: Execution roles for EventBridge and SSM

### Monitoring Components
- **CloudWatch Alarm**: `EC2SerialConsole-SSMAutomationFailures`
- **CloudWatch Alarm**: `EC2SerialConsole-EventBridgeFailures`

## 🧪 Testing the Solution

### 1. Test Detection
```bash
# Enable serial console (this should trigger the monitoring)
aws ec2 enable-serial-console-access

# Check if EventBridge detected the event (wait 5-15 minutes)
aws cloudwatch get-metric-statistics \
  --namespace AWS/Events \
  --metric-name MatchedEvents \
  --dimensions Name=RuleName,Value=EC2SerialConsoleEnableDetection \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 \
  --statistics Sum
```

### 2. Verify Remediation
```bash
# Check if serial console was automatically disabled
aws ec2 get-serial-console-access-status

# Check SSM automation execution
aws ssm describe-automation-executions \
  --filters "Key=DocumentNamePrefix,Values=EC2SerialConsoleImmediateRemediation"
```

### 3. Manual Testing
```bash
# Manually trigger the SSM automation
aws ssm start-automation-execution \
  --document-name EC2SerialConsoleImmediateRemediation \
  --parameters "AutoRemediation=true"
```

## 📈 Monitoring and Troubleshooting

### Key Metrics to Monitor
```bash
# EventBridge matched events
aws cloudwatch get-metric-statistics \
  --namespace AWS/Events \
  --metric-name MatchedEvents \
  --dimensions Name=RuleName,Value=EC2SerialConsoleEnableDetection

# EventBridge failed invocations  
aws cloudwatch get-metric-statistics \
  --namespace AWS/Events \
  --metric-name FailedInvocations \
  --dimensions Name=RuleName,Value=EC2SerialConsoleEnableDetection

# SSM automation success/failure
aws ssm describe-automation-executions \
  --filters "Key=DocumentNamePrefix,Values=EC2SerialConsoleImmediateRemediation"
```

### Common Troubleshooting

#### No Events Detected
```bash
# 1. Verify CloudTrail is logging management events
aws cloudtrail get-event-selectors --trail-name your-trail-name

# 2. Check CloudTrail is active
aws cloudtrail get-trail-status --name your-trail-name
```

#### EventBridge Rule Not Triggering
- **Check EventBridge metrics**: Look for `MatchedEvents` and `FailedInvocations`
- **Verify IAM permissions**: EventBridge execution role needs SSM permissions
- **Test event pattern**: Use AWS Console Event Pattern testing

#### SSM Automation Failures
```bash
# Get execution details
aws ssm get-automation-execution --automation-execution-id <execution-id>

# Check step-by-step execution
aws ssm describe-automation-step-executions --automation-execution-id <execution-id>
```

#### No Email Notifications
- **Confirm SNS subscription**: Check email and confirm subscription
- **Verify SNS permissions**: SSM automation role needs SNS publish permissions
- **Check spam folder**: Security alerts might be filtered

## 💰 Cost Estimate

**Monthly costs (us-east-1, minimal usage):**
- EventBridge: ~$0.10 (minimal events)
- SSM Automation: ~$0.05 (few executions, first 100K steps free)
- SNS: ~$0.50 (notifications)
- CloudWatch Alarms: ~$0.60 (2 alarms)
- **Total: ~$1.25/month**

*Note: CloudTrail costs depend on your existing setup and are not included.*

## 🔒 Security Considerations

### IAM Principle of Least Privilege
- EventBridge role: Only SSM execution permissions
- SSM role: Only EC2 serial console and SNS permissions
- No broad administrative permissions granted

### Event Security
- Events are processed through AWS managed services
- No sensitive data stored in Lambda functions (N/A - uses SSM)
- CloudTrail provides complete audit trail

### Network Security
- All communication over AWS internal networks
- No external network calls required
- SNS uses AWS managed encryption

## 🔧 Customization Options

### Notification Channels
Replace SNS email with:
- **Slack**: Use SNS → Lambda → Slack webhook
- **Microsoft Teams**: Use SNS → Lambda → Teams webhook  
- **AWS Security Hub**: Modify SSM to send findings
- **PagerDuty**: Use SNS → PagerDuty integration

### Additional Actions
Extend SSM automation to:
- Tag instances for tracking
- Create Security Hub findings
- Log to CloudWatch Logs
- Integrate with SIEM systems


## 🚨 Limitations

1. **CloudTrail Dependency**: Requires CloudTrail to be properly configured
2. **Event Delay**: ~5 minute average delay (1-15 minute range) due to CloudTrail processing
3. **API Calls Only**: Only detects actual `EnableSerialConsoleAccess` calls
. **No Policy Prevention**: Doesn't prevent IAM policy changes that grant permissions

## 🧹 Cleanup

```bash
# Delete the stack
aws cloudformation delete-stack --stack-name ec2-serial-console-event-monitoring

# Verify deletion
aws cloudformation describe-stacks --stack-name ec2-serial-console-event-monitoring
```

## 🆘 Support

### Getting Help
1. Check CloudWatch metrics and logs
2. Review SSM automation execution history
3. Verify CloudTrail configuration and status
4. Test EventBridge rule with sample events

### Common Event Pattern for Testing
```json
{
  "version": "0",
  "id": "test-event",
  "detail-type": "AWS API Call via CloudTrail",
  "source": "aws.ec2",
  "time": "2025-01-01T12:00:00Z",
  "detail": {
    "eventSource": "ec2.amazonaws.com",
    "eventName": "EnableSerialConsoleAccess",
    "userIdentity": {
      "accountId": "123456789012"
    },
    "awsRegion": "us-east-1",
    "sourceIPAddress": "203.0.113.1",
    "userAgent": "aws-cli/2.0.0"
  }
}
```

---

## �️ Alternative Security Approaches

While this event-driven solution provides excellent **reactive security** (detecting and responding to events), organizations should consider a **defense-in-depth approach** combining multiple strategies:

### **1. Service Control Policies (SCPs) - Preventive Control**

**Best Approach: Use SCPs to prevent unauthorized enablement entirely**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEC2SerialConsoleAccess",
      "Effect": "Deny",
      "Action": [
        "ec2:EnableSerialConsoleAccess",
        "ec2:GetSerialConsoleAccessStatus"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalTag/SerialConsoleAdmin": "true"
        }
      }
    }
  ]
}
```

**Advantages:**
- ✅ **Prevention**: Blocks the action before it happens (0-second response)
- ✅ **Organization-wide**: Applies across all accounts automatically
- ✅ **No infrastructure**: No resources to deploy or maintain
- ✅ **Cost**: Free (no additional AWS charges)


### **2. IAM Permission Boundaries - Account-Level Control**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAllExceptSerialConsole",
      "Effect": "Allow", 
      "NotAction": [
        "ec2:EnableSerialConsoleAccess",
        "ec2:DisableSerialConsoleAccess"
      ],
      "Resource": "*"
    }
  ]
}
```

**Permission Boundary Advantages:**
- ✅ **Account-specific**: Fine-tuned control per account
- ✅ **User/Role specific**: Can apply to specific identities
- ✅ **Break-glass friendly**: Easier to implement emergency procedures

### **3. AWS Config Rules - Compliance Monitoring**


**Config Rule Advantages:**
- ✅ **Continuous compliance**: Ongoing status monitoring
- ✅ **Dashboard visibility**: Clear compliance posture view
- ✅ **Integration**: Works with Security Hub, Systems Manager

**Config Rule Limitations:**
- ❌ **Slow response**: 1-24 hour detection time
- ❌ **Higher cost**: ~$2.70/month per rule
- ❌ **No prevention**: Only detects after the fact

### **4. Custom Lambda Solutions**

```python
# Lambda function triggered by CloudWatch Events
def lambda_handler(event, context):
    if event['detail']['eventName'] == 'EnableSerialConsoleAccess':
        # Custom response logic
        disable_serial_console()
        send_slack_notification()
```

**Lambda Advantages:**
- ✅ **Full customization**: Any response logic possible
- ✅ **Multiple integrations**: Slack, Teams, PagerDuty, etc.
- ✅ **Advanced filtering**: Complex event processing

**Lambda Limitations:**
- ❌ **Maintenance overhead**: Code updates, monitoring
- ❌ **Cold starts**: Potential response delays
- ❌ **Higher cost**: ~$2.80/month with compute costs

## 🏆 **Recommended Defense-in-Depth Strategy**

### **Tier 1: Prevention (Highest Priority)**
```
Service Control Policies (SCPs)
└── Block EnableSerialConsoleAccess for most users
└── Allow only for tagged emergency-access roles
```

### **Tier 2: Detection & Response (This Solution)**
```
Event-Driven Monitoring
└── Catch any successful enablement attempts
└── Immediate automated remediation
└── Security team notification
```

### **Tier 3: Compliance & Audit**
```
AWS Config Rules
└── Ongoing compliance monitoring
└── Security Hub integration
└── Historical compliance reporting
```

### **Implementation Priority:**

1. **Start with SCPs** (if using AWS Organizations)
   - Prevent the problem at the source
   - Zero additional cost
   - Organization-wide coverage

2. **Deploy Event-Driven Monitoring** (this solution)
   - Catch any bypass attempts or misconfigurations
   - Fast response time (~5 minutes)
   - Low cost (~$1.25/month)

3. **Add Config Rules** (for compliance)
   - Continuous compliance monitoring
   - Integration with security dashboards
   - Audit trail for reporting

**💡 Pro Tip**: Most organizations should implement SCPs first, then add this event-driven solution as a safety net. The combination provides both prevention and detection with minimal cost and complexity.

---

## 📖 Related Documentation

- [AWS CloudTrail Documentation](https://docs.aws.amazon.com/cloudtrail/)
- [Amazon EventBridge Documentation](https://docs.aws.amazon.com/eventbridge/)
- [AWS Systems Manager Automation](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html)
- [EC2 Serial Console Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-serial-console.html)
- [AWS Organizations SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
- [AWS Config Rules](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config.html)
- [AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html)

