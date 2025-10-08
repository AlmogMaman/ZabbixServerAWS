# 🚀 AWS CloudFormation Deployment Guide - Zabbix Monitoring System

## 📋 Overview

This CloudFormation template deploys a complete Zabbix 7.0 LTS monitoring infrastructure on AWS, replacing the VirtualBox setup with cloud-native services.

### What Gets Deployed

```
AWS Cloud
├── VPC (10.0.0.0/16)
│   ├── Public Subnets (2 AZs)
│   │   ├── Zabbix Server EC2 (Ubuntu 24.04)
│   │   └── Windows Server EC2 (Server 2022)
│   ├── Private Subnets (2 AZs)
│   │   └── RDS PostgreSQL 16
│   ├── Internet Gateway
│   ├── NAT Gateway
│   └── Security Groups
├── Elastic IPs
└── IAM Roles & Policies
```

## 🎯 Architecture Comparison

| Component | VirtualBox (Original) | AWS CloudFormation |
|-----------|----------------------|-------------------|
| **Zabbix Server** | VM in VirtualBox | EC2 t3.medium |
| **Database** | Local PostgreSQL | RDS PostgreSQL 16 |
| **Network** | Bridged/NAT | VPC with public/private subnets |
| **Windows Target** | Separate VM | EC2 Windows Server 2022 |
| **Storage** | VDI disk | EBS gp3 volumes |
| **Grafana** | Local installation | Same EC2 as Zabbix |
| **High Availability** | Single VM | Multi-AZ RDS, NAT Gateway |
| **Backup** | Manual snapshots | Automated RDS backups |
| **Security** | Host firewall | Security Groups, IAM |
| **Monitoring** | Manual | CloudWatch integration |

## 📦 Prerequisites

### Required Tools
- AWS CLI v2 installed and configured
- AWS Account with appropriate permissions
- EC2 Key Pair created in your target region
- Basic knowledge of AWS services

### Required Permissions
Your IAM user/role needs permissions for:
- EC2 (instances, VPC, security groups)
- RDS (database instances)
- IAM (roles, policies, instance profiles)
- CloudFormation (stack operations)
- Systems Manager (SSM for instance management)

### Cost Estimate
**Monthly cost estimate (us-east-1):**
- EC2 t3.medium (Zabbix): ~$30
- EC2 t3.medium (Windows): ~$30
- RDS db.t3.micro: ~$15
- NAT Gateway: ~$32
- EBS Storage (80GB): ~$8
- Data Transfer: ~$5-20
- **Total: ~$120-135/month**

💡 **Cost Optimization Tips:**
- Use t3a instances (10% cheaper)
- Stop Windows instance when not testing
- Use Single-AZ RDS for dev/test
- Remove NAT Gateway if not needed

## 🚀 Deployment Steps

### Step 1: Prepare Parameters

Create a parameters file `parameters.json`:

```json
[
  {
    "ParameterKey": "KeyPairName",
    "ParameterValue": "your-key-pair-name"
  },
  {
    "ParameterKey": "SSHAccessCIDR",
    "ParameterValue": "YOUR.IP.ADDRESS/32"
  },
  {
    "ParameterKey": "WebAccessCIDR",
    "ParameterValue": "0.0.0.0/0"
  },
  {
    "ParameterKey": "DBPassword",
    "ParameterValue": "YourSecureDBPassword123!"
  },
  {
    "ParameterKey": "ZabbixAdminPassword",
    "ParameterValue": "YourZabbixPassword123!"
  },
  {
    "ParameterKey": "GrafanaAdminPassword",
    "ParameterValue": "YourGrafanaPassword123!"
  },
  {
    "ParameterKey": "Environment",
    "ParameterValue": "Production"
  }
]
```

**Security Best Practices:**
- Never commit passwords to version control
- Use AWS Secrets Manager for production
- Replace `0.0.0.0/0` with specific IP ranges
- Enable MFA for AWS console access

### Step 2: Validate Template

```bash
# Validate the CloudFormation template
aws cloudformation validate-template \
  --template-body file://1-zabbix-main-stack.yaml

# Expected output: Template validation successful
```

### Step 3: Deploy Stack

```bash
# Deploy the stack
aws cloudformation create-stack \
  --stack-name zabbix-monitoring-prod \
  --template-body file://1-zabbix-main-stack.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Monitor stack creation
aws cloudformation describe-stacks \
  --stack-name zabbix-monitoring-prod \
  --query 'Stacks[0].StackStatus'
```

**Deployment Time:** 15-20 minutes

### Step 4: Monitor Deployment

```bash
# Watch stack events in real-time
aws cloudformation describe-stack-events \
  --stack-name zabbix-monitoring-prod \
  --max-items 10

# Or use AWS Console
# CloudFormation → Stacks → zabbix-monitoring-prod → Events
```

### Step 5: Retrieve Outputs

```bash
# Get all stack outputs
aws cloudformation describe-stacks \
  --stack-name zabbix-monitoring-prod \
  --query 'Stacks[0].Outputs'

# Get specific output (Zabbix URL)
aws cloudformation describe-stacks \
  --stack-name zabbix-monitoring-prod \
  --query 'Stacks[0].Outputs[?OutputKey==`ZabbixWebURL`].OutputValue' \
  --output text
```

## 🔧 Post-Deployment Configuration

### 1. Access Zabbix Web Interface

```bash
# Get Zabbix URL from outputs
ZABBIX_URL=$(aws cloudformation describe-stacks \
  --stack-name zabbix-monitoring-prod \
  --query 'Stacks[0].Outputs[?OutputKey==`ZabbixWebURL`].OutputValue' \
  --output text)

echo "Zabbix Web Interface: $ZABBIX_URL"
```

Open in browser and complete setup:
1. Navigate to the URL
2. Follow setup wizard (database already configured)
3. Login with: **Admin** / **zabbix**
4. **IMPORTANT:** Change default password immediately!
   - Administration → Users → Admin → Change password

### 2. Configure Windows Host Monitoring

The Windows agent is automatically installed, but you need to add it in Zabbix:

1. Login to Zabbix web interface
2. Go to **Data collection** → **Hosts**
3. Click **Create host**
4. Configure:
   - **Host name**: `Windows-Client`
   - **Groups**: Select or create `Windows servers`
   - **Interfaces**: 
     - Type: Agent
     - IP: Get from CloudFormation outputs `WindowsServerPrivateIP`
     - Port: `10050`
   - **Templates**: Link `Windows by Zabbix agent`
5. Click **Add**

Wait 1-2 minutes, then verify:
- Host status shows green "Available"

### 3. Add Disk Space Monitoring

**Create Items:**

1. Go to **Data collection** → **Hosts** → **Windows-Client** → **Items**
2. Click **Create item**

**Item 1: Free Space (Bytes)**
```
Name: Free disk space on C:
Type: Zabbix agent
Key: vfs.fs.size[C:,free]
Type of information: Numeric (unsigned)
Units: B
Update interval: 1m
```

**Item 2: Free Space (Percentage)**
```
Name: Free disk space on C: (percentage)
Type: Zabbix agent
Key: vfs.fs.size[C:,pfree]
Type of information: Numeric (float)
Units: %
Update interval: 1m
```

**Create Trigger:**
```
Name: Low disk space on C: drive
Severity: Warning
Expression: last(/Windows-Client/vfs.fs.size[C:,pfree])<20
```

### 4. Add Windows Audio Service Monitoring

**Create Item:**
```
Name: Windows Audio Service Status
Type: Zabbix agent
Key: service.audiosrv.running
Type of information: Numeric (unsigned)
Update interval: 30s
Show value: As is
Value mapping: 
  0 → Service stopped
  1 → Service running
```

**Create Trigger:**
```
Name: Windows Audio Service is down
Severity: High
Expression: last(/Windows-Client/service.audiosrv.running)=0
Description: Windows Audio Service (Audiosrv) is not running
```

### 5. Add Cloudflare DNS Monitoring

1. Go to **Data collection** → **Hosts** → **Zabbix server**
2. Click **Items** → **Create item**

**Item 1: ICMP Ping**
```
Name: Cloudflare DNS (1.1.1.1) ICMP ping
Type: Simple check
Key: icmpping[1.1.1.1]
Type of information: Numeric (unsigned)
Update interval: 30s
```

**Item 2: Response Time**
```
Name: Cloudflare DNS (1.1.1.1) response time
Type: Simple check
Key: icmppingsec[1.1.1.1]
Type of information: Numeric (float)
Units: s
Update interval: 30s
```

**Create Trigger:**
```
Name: Cloudflare DNS is unreachable
Severity: Average
Expression: last(/Zabbix server/icmpping[1.1.1.1])=0
```

### 6. Configure Grafana

```bash
# Get Grafana URL
GRAFANA_URL=$(aws cloudformation describe-stacks \
  --stack-name zabbix-monitoring-prod \
  --query 'Stacks[0].Outputs[?OutputKey==`GrafanaURL`].OutputValue' \
  --output text)

echo "Grafana URL: $GRAFANA_URL"
```

**Setup Steps:**

1. Open Grafana URL in browser
2. Login with credentials from stack outputs
3. Enable Zabbix plugin:
   - Configuration → Plugins
   - Search "Zabbix"
   - Enable the plugin

4. Add Zabbix Data Source:
   - Configuration → Data Sources → Add data source
   - Select **Zabbix**
   - Configure:
     ```
     Name: Zabbix-Server
     URL: http://localhost/api_jsonrpc.php
     Username: Admin
     Password: [your Zabbix password]
     ```
   - Click **Save & Test**

5. Create Dashboard:
   - Dashboards → New Dashboard
   - Add panels for:
     - Disk Space (Time series)
     - Audio Service Status (Stat)
     - Cloudflare DNS Response (Time series)
     - Active Problems (Table)

## 📊 Verification Checklist

Run these checks to verify your deployment:

```bash
# 1. Check stack status
aws cloudformation describe-stacks \
  --stack-name zabbix-monitoring-prod \
  --query 'Stacks[0].StackStatus'
# Expected: CREATE_COMPLETE

# 2. Check Zabbix server is running
ZABBIX_IP=$(aws cloudformation describe-stacks \
  --stack-name zabbix-monitoring-prod \
  --query 'Stacks[0].Outputs[?OutputKey==`ZabbixServerPublicIP`].OutputValue' \
  --output text)

curl -s http://$ZABBIX_IP | grep -i zabbix
# Expected: HTML containing "Zabbix"

# 3. Check Grafana is running
curl -s http://$ZABBIX_IP:3000 | grep -i grafana
# Expected: HTML containing "Grafana"

# 4. Test SSH access
ssh -i your-key.pem ubuntu@$ZABBIX_IP "systemctl status zabbix-server"
# Expected: active (running)

# 5. Check database connectivity
ssh -i your-key.pem ubuntu@$ZABBIX_IP \
  "PGPASSWORD='YourDBPassword' psql -h [RDS-ENDPOINT] -U zabbix -d zabbix -c 'SELECT version();'"
# Expected: PostgreSQL version output
```

**Manual Verification:**

- [ ] Zabbix web interface accessible
- [ ] Zabbix server status: Running (green)
- [ ] Windows client status: Available (green)
- [ ] Disk space items collecting data
- [ ] Audio service monitoring active
- [ ] Cloudflare DNS monitoring active
- [ ] Grafana accessible
- [ ] Zabbix data source connected in Grafana
- [ ] Dashboard displaying metrics

## 🔐 Security Hardening

### 1. Update Security Groups

Restrict access to known IPs only:

```bash
# Update Zabbix server SG to allow SSH from your IP only
aws ec2 authorize-security-group-ingress \
  --group-id [ZABBIX-SG-ID] \
  --protocol tcp \
  --port 22 \
  --cidr YOUR.IP.ADDRESS/32

# Remove the broad 0.0.0.0/0 rule
aws ec2 revoke-security-group-ingress \
  --group-id [ZABBIX-SG-ID] \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

### 2. Enable HTTPS

Install Let's Encrypt SSL certificate:

```bash
# SSH to Zabbix server
ssh -i your-key.pem ubuntu@$ZABBIX_IP

# Install certbot
sudo apt install -y certbot python3-certbot-nginx

# Get certificate (requires domain name)
sudo certbot --nginx -d your-domain.com

# Auto-renewal is configured automatically
sudo certbot renew --dry-run
```

### 3. Enable RDS Encryption at Rest

For production, use encrypted RDS (already enabled in template).

### 4. Configure CloudWatch Alarms

```bash
# Create CPU alarm for Zabbix server
aws cloudwatch put-metric-alarm \
  --alarm-name zabbix-high-cpu \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=[INSTANCE-ID]
```

### 5. Enable AWS Systems Manager Session Manager

Access instances without opening SSH ports:

```bash
# Start session (no key required)
aws ssm start-session --target [INSTANCE-ID]

# This uses the IAM roles already configured in the template
```

## 🔄 Updates and Maintenance

### Update Stack

```bash
# Update stack with new parameters or template changes
aws cloudformation update-stack \
  --stack-name zabbix-monitoring-prod \
  --template-body file://1-zabbix-main-stack.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

### Backup Strategy

**Automated Backups (Already Configured):**
- RDS automated backups: 7 days retention
- Daily backup window: 03:00-04:00 UTC
- Maintenance window: Monday 04:00-05:00 UTC

**Manual Backups:**

```bash
# Create RDS snapshot
aws rds create-db-snapshot \
  --db-instance-identifier zabbix-monitoring-prod-zabbix-db \
  --db-snapshot-identifier zabbix-manual-backup-$(date +%Y%m%d)

# Create EC2 AMI
INSTANCE_ID=$(aws cloudformation describe-stack-resources \
  --stack-name zabbix-monitoring-prod \
  --logical-resource-id ZabbixServerInstance \
  --query 'StackResources[0].PhysicalResourceId' \
  --output text)

aws ec2 create-image \
  --instance-id $INSTANCE_ID \
  --name "zabbix-server-backup-$(date +%Y%m%d)" \
  --no-reboot
```

### Restore from Backup

```bash
# Restore RDS from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier zabbix-db-restored \
  --db-snapshot-identifier zabbix-manual-backup-20251008

# Update Zabbix server configuration to point to new DB
ssh -i your-key.pem ubuntu@$ZABBIX_IP
sudo nano /etc/zabbix/zabbix_server.conf
# Update DBHost=<new-rds-endpoint>
sudo systemctl restart zabbix-server
```

## 📈 Scaling Considerations

### Vertical Scaling

Increase instance sizes:

```bash
# Stop instance
aws ec2 stop-instances --instance-ids [INSTANCE-ID]

# Change instance type
aws ec2 modify-instance-attribute \
  --instance-id [INSTANCE-ID] \
  --instance-type t3.large

# Start instance
aws ec2 start-instances --instance-ids [INSTANCE-ID]
```

### Horizontal Scaling

For high availability:
1. Deploy Zabbix proxies in multiple regions
2. Use RDS Multi-AZ deployment
3. Add Application Load Balancer
4. Deploy multiple Zabbix servers (requires clustering)

## 🧹 Cleanup

### Delete Stack

```bash
# Delete the entire infrastructure
aws cloudformation delete-stack \
  --stack-name zabbix-monitoring-prod

# Monitor deletion
aws cloudformation wait stack-delete-complete \
  --stack-name zabbix-monitoring-prod

echo "Stack deleted successfully"
```

**Note:** RDS has DeletionPolicy: Snapshot, so a final snapshot will be created before deletion.

### Delete Snapshots

```bash
# List snapshots
aws rds describe-db-snapshots \
  --query 'DBSnapshots[?contains(DBSnapshotIdentifier,`zabbix`)].DBSnapshotIdentifier'

# Delete specific snapshot
aws rds delete-db-snapshot \
  --db-snapshot-identifier [SNAPSHOT-ID]
```

## 🐛 Troubleshooting

### Stack Creation Failed

```bash
# Check events for error details
aws cloudformation describe-stack-events \
  --stack-name zabbix-monitoring-prod \
  --max-items 20

# Common issues:
# 1. KeyPair not found → Create EC2 key pair first
# 2. Limit exceeded → Request limit increase
# 3. Subnet conflicts → Adjust CIDR blocks
```

### Zabbix Server Not Starting

```bash
# SSH to server
ssh -i your-key.pem ubuntu@$ZABBIX_IP

# Check logs
sudo tail -f /var/log/zabbix/zabbix_server.log

# Check service status
sudo systemctl status zabbix-server

# Common issues:
# 1. Database connection → Check RDS security group
# 2. Schema import failed → Manual import required
```

### Windows Agent Not Connecting

```bash
# From Zabbix server, test connectivity
zabbix_get -s [WINDOWS-PRIVATE-IP] -k agent.ping

# On Windows (via RDP), check:
# 1. Service status: Get-Service "Zabbix Agent"
# 2. Firewall rule: Get-NetFirewallRule -DisplayName "Zabbix Agent"
# 3. Agent logs: C:\Program Files\Zabbix Agent\zabbix_agentd.log
```

### Can't Access Web Interface

```bash
# Check security group rules
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=*Zabbix-Server-SG*" \
  --query 'SecurityGroups[0].IpPermissions'

# Check NGINX status
ssh -i your-key.pem ubuntu@$ZABBIX_IP \
  "sudo systemctl status nginx"

# Test locally
ssh -i your-key.pem ubuntu@$ZABBIX_IP \
  "curl -I http://localhost"
```

## 📚 Additional Resources

### AWS Documentation
- [CloudFormation User Guide](https://docs.aws.amazon.com/cloudformation/)
- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [RDS User Guide](https://docs.aws.amazon.com/rds/)
- [VPC User Guide](https://docs.aws.amazon.com/vpc/)

### Zabbix Documentation
- [Official Documentation](https://www.zabbix.com/documentation/current)
- [API Documentation](https://www.zabbix.com/documentation/current/manual/api)
- [Template Library](https://www.zabbix.com/integrations)

### Useful Commands

```bash
# Get all running costs
aws ce get-cost-and-usage \
  --time-period Start=2025-10-01,End=2025-10-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=TAG,Key=aws:cloudformation:stack-name

# List all resources in stack
aws cloudformation list-stack-resources \
  --stack-name zabbix-monitoring-prod

# Export stack template
aws cloudformation get-template \
  --stack-name zabbix-monitoring-prod \
  --query TemplateBody \
  --output text > exported-template.yaml
```

## 🎓 Learning Outcomes

After completing this deployment, you will understand:

✅ AWS VPC networking and subnets  
✅ EC2 instance deployment and configuration  
✅ RDS PostgreSQL database setup  
✅ Security Groups and network security  
✅ IAM roles and instance profiles  
✅ CloudFormation Infrastructure as Code  
✅ User Data scripts for automation  
✅ Multi-tier application architecture  
✅ Monitoring and observability setup  
✅ Cost optimization in AWS  

---

**Deployment Time:** 20 minutes  
**Estimated Monthly Cost:** $120-135  
**Difficulty:** Intermediate  
**Last Updated:** October 2025
