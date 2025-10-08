# 🔍 Zabbix Monitoring System - Complete Implementation Guide

## 📋 Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Step-by-Step Installation](#step-by-step-installation)
  - [Part 1: VirtualBox Setup](#part-1-virtualbox-setup)
  - [Part 2: Ubuntu Server Installation](#part-2-ubuntu-server-installation)
  - [Part 3: Zabbix Server Installation](#part-3-zabbix-server-installation)
  - [Part 4: Windows Agent Setup](#part-4-windows-agent-setup)
  - [Part 5: Monitoring Configuration](#part-5-monitoring-configuration)
  - [Part 6: Grafana Dashboard](#part-6-grafana-dashboard)
- [Troubleshooting](#troubleshooting)
- [Additional Resources](#additional-resources)

---

## 🎯 Overview

This project implements a comprehensive monitoring solution using **Zabbix 7.0 LTS** on **Ubuntu Server 24.04 LTS** running in VirtualBox. The system monitors:

- ✅ **Disk Space** on Windows machine
- ✅ **Windows Audio Service** status
- ✅ **Cloudflare DNS** (1.1.1.1) connectivity
- ✅ **Custom Dashboard** with Grafana integration
- ✅ **Automated Alerts**

### Technology Stack:
- **OS**: Ubuntu Server 24.04 LTS
- **Monitoring**: Zabbix 7.0 LTS
- **Database**: PostgreSQL 16
- **Web Server**: NGINX
- **Visualization**: Grafana
- **Virtualization**: Oracle VirtualBox

---

## 💻 Prerequisites

### Required Software:
- **VirtualBox** 7.0+ ([Download](https://www.virtualbox.org/wiki/Downloads))
- **Ubuntu Server 24.04 LTS ISO** ([Download](https://ubuntu.com/download/server))
- **Windows Machine** (physical or VM) for monitoring
- Minimum **4GB RAM** and **20GB disk space**

### Network Requirements:
- Internet connection for package installation
- Network connectivity between Zabbix server and monitored hosts

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│           VirtualBox Host Machine            │
│                                              │
│  ┌────────────────────────────────────┐    │
│  │   Ubuntu Server 24.04 LTS VM       │    │
│  │   IP: 192.168.1.100 (example)      │    │
│  │                                     │    │
│  │   ┌──────────────────────┐         │    │
│  │   │   Zabbix Server      │         │    │
│  │   │   - PostgreSQL DB    │         │    │
│  │   │   - NGINX Web        │         │    │
│  │   │   - Zabbix Agent     │         │    │
│  │   └──────────────────────┘         │    │
│  │                                     │    │
│  │   ┌──────────────────────┐         │    │
│  │   │   Grafana            │         │    │
│  │   │   Port: 3000         │         │    │
│  │   └──────────────────────┘         │    │
│  └────────────────────────────────────┘    │
│                                              │
│  ┌────────────────────────────────────┐    │
│  │   Windows Machine/VM                │    │
│  │   - Zabbix Agent                    │    │
│  │   - Monitored Services              │    │
│  └────────────────────────────────────┘    │
│                                              │
│  External Monitoring:                       │
│  └─> Cloudflare DNS (1.1.1.1)              │
└─────────────────────────────────────────────┘
```

---

## 📦 Step-by-Step Installation

### Part 1: VirtualBox Setup

#### 1.1 Create New Virtual Machine

1. Open VirtualBox and click **"New"**
2. Configure the VM:
   - **Name**: `Zabbix-Server`
   - **Type**: Linux
   - **Version**: Ubuntu (64-bit)
   - **Memory**: 4096 MB (4GB)
   - **Hard Disk**: Create virtual hard disk (20GB, VDI, Dynamically allocated)

#### 1.2 Network Configuration

1. Go to VM **Settings** → **Network**
2. **Adapter 1**:
   - Enable Network Adapter
   - Attached to: **Bridged Adapter** (for LAN access)
   - Alternative: **NAT** with Port Forwarding:
     - SSH: Host Port 2222 → Guest Port 22
     - HTTP: Host Port 8080 → Guest Port 80
     - Zabbix: Host Port 10051 → Guest Port 10051
     - Grafana: Host Port 3000 → Guest Port 3000

#### 1.3 Mount Ubuntu ISO

1. Go to **Settings** → **Storage**
2. Click on **Empty** under Controller: IDE
3. Click disk icon → **Choose a disk file**
4. Select Ubuntu Server 24.04 LTS ISO

---

### Part 2: Ubuntu Server Installation

#### 2.1 Boot and Install Ubuntu

1. Start the VM
2. Select **"Install Ubuntu Server"**
3. Follow installation wizard:
   - Language: English
   - Keyboard: Your layout
   - Network: Configure DHCP (note the IP address)
   - Storage: Use entire disk
   - Profile Setup:
     - Your name: `admin`
     - Server name: `zabbix-server`
     - Username: `admin`
     - Password: *[your secure password]*
   - **Install OpenSSH server**: ✅ Yes
   - Featured Server Snaps: Skip

4. Wait for installation to complete
5. **Reboot** and remove installation media

#### 2.2 Initial System Update

```bash
# Login with your credentials
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y wget curl vim net-tools ufw

# Configure firewall
sudo ufw allow OpenSSH
sudo ufw allow http
sudo ufw allow https
sudo ufw allow 10050:10051/tcp
sudo ufw allow 3000/tcp
sudo ufw enable
```

---

### Part 3: Zabbix Server Installation

#### 3.1 Install PostgreSQL Database

```bash
# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Start and enable PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create Zabbix database and user
sudo -u postgres psql <<EOF
CREATE DATABASE zabbix;
CREATE USER zabbix WITH PASSWORD 'SecurePassword123!';
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
\c zabbix
GRANT ALL ON SCHEMA public TO zabbix;
ALTER DATABASE zabbix OWNER TO zabbix;
\q
EOF
```

#### 3.2 Install Zabbix Repository

```bash
# Download and install Zabbix repository package
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest+ubuntu24.04_all.deb
sudo apt update
```

#### 3.3 Install Zabbix Server Components

```bash
# Install Zabbix server, frontend, agent
sudo apt install -y zabbix-server-pgsql zabbix-frontend-php \
    php8.3-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent

# Import initial database schema
sudo zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | \
    sudo -u zabbix psql zabbix
```

#### 3.4 Configure Zabbix Server

```bash
# Edit Zabbix server configuration
sudo vim /etc/zabbix/zabbix_server.conf

# Find and update these lines:
# DBPassword=SecurePassword123!
# DBHost=localhost

# Use this sed command for quick update:
sudo sed -i 's/# DBPassword=/DBPassword=SecurePassword123!/' /etc/zabbix/zabbix_server.conf
```

#### 3.5 Configure NGINX

```bash
# Edit NGINX configuration for Zabbix
sudo vim /etc/zabbix/nginx.conf

# Uncomment and set:
# listen 80;
# server_name your_domain_or_ip;

# Quick update with sed:
sudo sed -i 's/# listen/listen/' /etc/zabbix/nginx.conf
sudo sed -i 's/# server_name/server_name/' /etc/zabbix/nginx.conf

# Edit PHP configuration
sudo vim /etc/zabbix/php-fpm.conf

# Update timezone (find php_value[date.timezone]):
# php_value[date.timezone] = Asia/Jerusalem

# Quick update:
sudo sed -i 's/;php_value\[date.timezone\]/php_value[date.timezone]/' /etc/zabbix/php-fpm.conf
sudo sed -i 's/Europe\/Riga/Asia\/Jerusalem/' /etc/zabbix/php-fpm.conf
```

#### 3.6 Start Zabbix Services

```bash
# Restart and enable all services
sudo systemctl restart zabbix-server zabbix-agent nginx php8.3-fpm
sudo systemctl enable zabbix-server zabbix-agent nginx php8.3-fpm

# Check service status
sudo systemctl status zabbix-server
sudo systemctl status nginx
```

#### 3.7 Complete Web Setup

1. Open browser and navigate to: `http://[SERVER_IP]`
2. Zabbix setup wizard:
   - Welcome: Click **Next**
   - Pre-requisites: Verify all OK → **Next**
   - Database connection:
     - Type: PostgreSQL
     - Host: localhost
     - Port: 5432
     - Database: zabbix
     - User: zabbix
     - Password: SecurePassword123!
   - Server details: Keep defaults → **Next**
   - Pre-installation summary: **Next**
   - Install: **Finish**

3. **Login**:
   - Username: `Admin`
   - Password: `zabbix`

4. **Change default password** (Recommended):
   - Go to **Administration** → **Users** → **Admin**
   - Click **Change password**

---

### Part 4: Windows Agent Setup

#### 4.1 Download Zabbix Agent

On your Windows machine:

1. Download Zabbix Agent from: https://www.zabbix.com/download_agents
2. Select:
   - Version: 7.0 LTS
   - OS: Windows (64-bit/32-bit based on your system)
3. Download the MSI installer

#### 4.2 Install Zabbix Agent

1. Run the MSI installer as Administrator
2. Installation wizard:
   - Server: `[ZABBIX_SERVER_IP]` (your Ubuntu VM IP)
   - ServerActive: `[ZABBIX_SERVER_IP]`
   - Hostname: `Windows-Client` (or your computer name)
   - Agent service: ✅ Install as Windows service

3. Complete installation

#### 4.3 Configure Agent for Service Monitoring

Edit the agent configuration:

```batch
# Open PowerShell as Administrator
notepad "C:\Program Files\Zabbix Agent\zabbix_agentd.conf"

# Add these lines at the end:
UserParameter=service.audiosrv.state,powershell -NoProfile -Command "Get-Service -Name 'Audiosrv' | Select-Object -ExpandProperty Status"
UserParameter=service.audiosrv.running,powershell -NoProfile -Command "if ((Get-Service -Name 'Audiosrv').Status -eq 'Running') { 1 } else { 0 }"

# Save and restart the service
Restart-Service "Zabbix Agent"
```

#### 4.4 Verify Agent

```batch
# Check if service is running
Get-Service "Zabbix Agent"

# Test agent from server
# On Ubuntu VM:
zabbix_get -s [WINDOWS_IP] -k agent.ping
```

---

### Part 5: Monitoring Configuration

#### 5.1 Add Windows Host to Zabbix

1. Login to Zabbix web interface
2. Go to **Data collection** → **Hosts**
3. Click **Create host**
4. Configure:
   - **Host name**: `Windows-Client`
   - **Groups**: `Windows servers` (select or create)
   - **Interfaces**:
     - Add **Agent** interface
     - IP address: `[WINDOWS_IP]`
     - Port: `10050`
   - **Templates**:
     - Select **"Windows by Zabbix agent"**
   - Click **Add**

#### 5.2 Create Disk Space Monitoring

1. Go to **Data collection** → **Hosts** → **Windows-Client**
2. Click **Items** → **Create item**
3. Configure:
   - **Name**: `Free disk space on C:`
   - **Type**: Zabbix agent
   - **Key**: `vfs.fs.size[C:,free]`
   - **Type of information**: Numeric (unsigned)
   - **Units**: B
   - **Update interval**: 1m
   - Click **Add**

4. Create percentage item:
   - **Name**: `Free disk space on C: (percentage)`
   - **Type**: Zabbix agent
   - **Key**: `vfs.fs.size[C:,pfree]`
   - **Type of information**: Numeric (float)
   - **Units**: %
   - **Update interval**: 1m
   - Click **Add**

#### 5.3 Create Windows Audio Service Monitoring

1. Click **Items** → **Create item**
2. Configure:
   - **Name**: `Windows Audio Service Status`
   - **Type**: Zabbix agent
   - **Key**: `service.audiosrv.running`
   - **Type of information**: Numeric (unsigned)
   - **Update interval**: 30s
   - **Value mapping**: 
     - 0 → Not Running
     - 1 → Running
   - Click **Add**

3. Create trigger for alert:
   - Go to **Triggers** → **Create trigger**
   - **Name**: `Windows Audio Service is down`
   - **Severity**: High
   - **Expression**: `last(/Windows-Client/service.audiosrv.running)=0`
   - Click **Add**

#### 5.4 Create Cloudflare DNS Monitoring

1. Go to **Data collection** → **Hosts** → **Zabbix server**
2. Click **Items** → **Create item**
3. Configure:
   - **Name**: `Cloudflare DNS (1.1.1.1) ICMP ping`
   - **Type**: Simple check
   - **Key**: `icmpping[1.1.1.1]`
   - **Type of information**: Numeric (unsigned)
   - **Update interval**: 30s
   - Click **Add**

4. Create response time item:
   - **Name**: `Cloudflare DNS (1.1.1.1) ICMP response time`
   - **Type**: Simple check
   - **Key**: `icmppingsec[1.1.1.1]`
   - **Type of information**: Numeric (float)
   - **Units**: s
   - **Update interval**: 30s
   - Click **Add**

5. Create trigger:
   - Go to **Triggers** → **Create trigger**
   - **Name**: `Cloudflare DNS is unreachable`
   - **Severity**: Average
   - **Expression**: `last(/Zabbix server/icmpping[1.1.1.1])=0`
   - Click **Add**

#### 5.5 Create Native Zabbix Dashboard

1. Go to **Monitoring** → **Dashboards**
2. Click **Create dashboard**
3. **Name**: `System Monitoring Overview`
4. Click **Add widget**:

   **Widget 1: Graph - Disk Space**
   - Type: Graph
   - Name: `C: Drive Free Space`
   - Data set:
     - Host: Windows-Client
     - Item: `Free disk space on C: (percentage)`
   - Click **Add**

   **Widget 2: Plain text - Audio Service**
   - Type: Plain text
   - Name: `Windows Audio Service`
   - Items:
     - Host: Windows-Client
     - Item: `Windows Audio Service Status`
   - Click **Add**

   **Widget 3: Graph - Cloudflare DNS**
   - Type: Graph
   - Name: `Cloudflare DNS Response Time`
   - Data set:
     - Host: Zabbix server
     - Item: `Cloudflare DNS (1.1.1.1) ICMP response time`
   - Click **Add**

   **Widget 4: Problems - Triggers**
   - Type: Problems
   - Name: `Active Problems`
   - Show: Problems
   - Hosts: All
   - Click **Add**

5. **Save** dashboard

---

### Part 6: Grafana Dashboard

#### 6.1 Install Grafana

```bash
# Add Grafana repository
sudo apt install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Install Grafana
sudo apt update
sudo apt install -y grafana

# Start and enable Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

#### 6.2 Install Zabbix Plugin for Grafana

```bash
# Install Zabbix plugin
sudo grafana-cli plugins install alexanderzobnin-zabbix-app

# Restart Grafana
sudo systemctl restart grafana-server
```

#### 6.3 Configure Grafana

1. Open browser: `http://[SERVER_IP]:3000`
2. Login:
   - Username: `admin`
   - Password: `admin`
   - Change password when prompted

3. **Add Zabbix Data Source**:
   - Go to **Configuration** → **Data Sources**
   - Click **Add data source**
   - Select **Zabbix**
   - Configure:
     - Name: `Zabbix-Server`
     - URL: `http://localhost/api_jsonrpc.php`
     - Username: `Admin`
     - Password: `[your Zabbix password]`
   - Click **Save & Test**

#### 6.4 Create Grafana Dashboard

1. Go to **Dashboards** → **New Dashboard**
2. Click **Add visualization**
3. Select **Zabbix-Server** data source

**Panel 1: Disk Space**
- Title: `Windows C: Drive Free Space`
- Query:
  - Host: `Windows-Client`
  - Item: `Free disk space on C: (percentage)`
- Visualization: Time series
- Unit: Percent (0-100)

**Panel 2: Audio Service**
- Title: `Windows Audio Service Status`
- Query:
  - Host: `Windows-Client`
  - Item: `Windows Audio Service Status`
- Visualization: Stat
- Value mappings:
  - 0 = Stopped (red)
  - 1 = Running (green)

**Panel 3: Cloudflare DNS**
- Title: `Cloudflare DNS Response Time`
- Query:
  - Host: `Zabbix server`
  - Item: `Cloudflare DNS (1.1.1.1) ICMP response time`
- Visualization: Time series
- Unit: seconds (s)

**Panel 4: Problems**
- Title: `Active Triggers`
- Visualization: Logs
- Query: All active problems

4. **Save dashboard**: Name it `System Monitoring Dashboard`

---

## 🔧 Troubleshooting

### Zabbix Server Not Starting

```bash
# Check logs
sudo tail -f /var/log/zabbix/zabbix_server.log

# Common issues:
# 1. Database connection - verify PostgreSQL is running
sudo systemctl status postgresql

# 2. Check database configuration
sudo grep DBPassword /etc/zabbix/zabbix_server.conf
```

### Agent Connection Issues

```bash
# Test from server
zabbix_get -s [AGENT_IP] -k agent.ping

# Check firewall on Windows
# Run PowerShell as Admin:
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -LocalPort 10050 -Protocol TCP -Action Allow

# Check agent logs on Windows:
# C:\Program Files\Zabbix Agent\zabbix_agentd.log
```

### NGINX Configuration

```bash
# Test NGINX configuration
sudo nginx -t

# If port 80 is in use:
sudo netstat -tulpn | grep :80

# Check NGINX error logs
sudo tail -f /var/log/nginx/error.log
```

### Grafana Connection Issues

```bash
# Verify Grafana is running
sudo systemctl status grafana-server

# Check Grafana logs
sudo tail -f /var/log/grafana/grafana.log

# Reset admin password if needed
sudo grafana-cli admin reset-admin-password newpassword
```

---

## 📊 Verification Checklist

- [ ] Ubuntu Server 24.04 LTS installed in VirtualBox
- [ ] PostgreSQL database created and configured
- [ ] Zabbix 7.0 LTS server installed and running
- [ ] NGINX web server serving Zabbix frontend
- [ ] Zabbix web interface accessible
- [ ] Windows agent installed and communicating
- [ ] Disk space monitoring active
- [ ] Windows Audio Service monitoring active
- [ ] Cloudflare DNS monitoring active
- [ ] Triggers configured and working
- [ ] Grafana installed and configured
- [ ] Zabbix plugin integrated with Grafana
- [ ] Dashboard created and displaying data

---

## 📚 Additional Resources

### Official Documentation
- [Zabbix Documentation](https://www.zabbix.com/documentation/current)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [NGINX Documentation](https://nginx.org/en/docs/)
- [Grafana Documentation](https://grafana.com/docs/)

### Useful Commands

```bash
# Zabbix service management
sudo systemctl status zabbix-server
sudo systemctl restart zabbix-server
sudo tail -f /var/log/zabbix/zabbix_server.log

# Database access
sudo -u postgres psql -d zabbix

# Test agent connectivity
zabbix_get -s [HOST_IP] -k system.uname

# Check listening ports
sudo netstat -tulpn | grep -E '(80|10050|10051|3000)'
```

### Network Diagram

```
Internet
   ↓
Cloudflare DNS (1.1.1.1) ←─── ICMP Ping ───┐
                                             │
   ┌─────────────────────────────────────────┼──────┐
   │ VirtualBox Host Network                 │      │
   │                                          │      │
   │  ┌────────────────────┐                 │      │
   │  │ Ubuntu VM          │                 │      │
   │  │ Zabbix Server      │←── Agent ───────┘      │
   │  │ 192.168.1.100:80   │                        │
   │  │ NGINX + PostgreSQL │                        │
   │  │ Grafana :3000      │                        │
   │  └────────────────────┘                        │
   │           ↓                                     │
   │    Agent Protocol                               │
   │      (Port 10050)                               │
   │           ↓                                     │
   │  ┌────────────────────┐                        │
   │  │ Windows Machine    │                        │
   │  │ Zabbix Agent       │                        │
   │  │ - Disk Space       │                        │
   │  │ - Audio Service    │                        │
   │  └────────────────────┘                        │
   └────────────────────────────────────────────────┘
```

---

## 🎓 Assignment Compliance

This implementation fulfills all requirements:

✅ **VirtualBox Installation**: Ubuntu Server running in VirtualBox  
✅ **Ubuntu Server LTS**: Version 24.04 LTS  
✅ **Zabbix LTS**: Version 7.0 LTS  
✅ **PostgreSQL Database**: Used instead of MySQL  
✅ **NGINX Web Server**: Used instead of Apache  
✅ **Windows Monitoring**: Disk space + Audio service  
✅ **DNS Monitoring**: Cloudflare 1.1.1.1  
✅ **Dashboard**: Both Zabbix native and Grafana  
✅ **No Appliance**: Manual installation from scratch  
✅ **Multiple Servers**: Separate Windows machine monitored  

---

## 👨‍💻 Author Notes

This project demonstrates:
- Linux server administration
- Database configuration (PostgreSQL)
- Web server setup (NGINX)
- Monitoring system implementation
- Network connectivity troubleshooting
- Windows-Linux integration
- Dashboard visualization

**Time to Complete**: 2-3 hours  
**Difficulty Level**: Intermediate  
**Learning Outcomes**: Production-ready monitoring system

---

## 📝 License

This documentation is provided as-is for educational purposes.

---

**Last Updated**: October 2025  
**Zabbix Version**: 7.0 LTS  
**Ubuntu Version**: 24.04 LTS  
**Tested On**: VirtualBox 7.0
