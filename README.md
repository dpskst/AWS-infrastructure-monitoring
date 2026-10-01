# Project 07 — AWS Infrastructure Monitoring

## 1. Project Overview

AWS 환경에서 **Zabbix + Amazon CloudWatch + Grafana**를 연계하여 서버 및 AWS 인프라 모니터링 환경을 구축합니다.

- Zabbix를 이용한 Linux 서버 리소스 모니터링
- Amazon CloudWatch를 이용한 AWS EC2 모니터링
- Grafana를 이용한 통합 모니터링 대시보드 구성
- AWS VPC / Public Subnet / Private Subnet 기반의 네트워크 구성
- Security Group을 이용한 모니터링 서버와 대상 서버 간 접근 제어
- AWS Systems Manager Session Manager를 이용한 관리 환경 구성

---

## 2. Project Architecture

```text
                         Internet
                            │
                            ▼
                    ┌─────────────────┐
                    │ Internet Gateway│
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Public Subnet  │
                    │  10.0.1.0/24    │
                    │   us-east-2a    │
                    └────────┬────────┘
                             │
                ┌────────────▼────────────┐
                │ project07-zabbix-server │
                │ Ubuntu 24.04 LTS        │
                │ t3.small                │
                │                         │
                │ Zabbix Server           │
                │ PostgreSQL              │
                │ Nginx                   │
                │ Grafana                 │
                └────────────┬────────────┘
                             │
                      Zabbix Agent
                       TCP 10050
                             │
                    ┌────────▼────────┐
                    │ Private Subnet  │
                    │  10.0.2.0/24    │
                    │   us-east-2a    │
                    └────────┬────────┘
                             │
                ┌────────────▼──────────────┐
                │ project07-monitoring-target│
                │ Ubuntu 24.04 LTS           │
                │ t3.micro                   │
                │                            │
                │ Zabbix Agent               │
                └────────────────────────────┘


       AWS EC2 Metrics
              │
              ▼
      ┌─────────────────┐
      │ Amazon CloudWatch│
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │     Grafana     │
      │ Unified Dashboard│
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │      Zabbix     │
      │ Linux Monitoring│
      └─────────────────┘
```

---

## 3. Project Environment

### AWS

| 항목 | 구성 |
|---|---|
| Region | us-east-2 (Ohio) |
| VPC | project07-vpc |
| VPC CIDR | 10.0.0.0/16 |
| Public Subnet | project07-public-subnet |
| Public CIDR | 10.0.1.0/24 |
| Private Subnet | project07-private-subnet |
| Private CIDR | 10.0.2.0/24 |
| Availability Zone | us-east-2a |
| Internet Gateway | project07-igw |
| NAT Gateway | project07-nat |

### Monitoring Server

| 항목 | 구성 |
|---|---|
| Name | project07-zabbix-server |
| OS | Ubuntu Server 24.04 LTS |
| Instance Type | t3.small |
| Private IP | 10.0.1.194 |
| Service | Zabbix Server / PostgreSQL / Nginx / Grafana |

### Monitoring Target

| 항목 | 구성 |
|---|---|
| Name | project07-monitoring-target |
| OS | Ubuntu Server 24.04 LTS |
| Instance Type | t3.micro |
| Private IP | 10.0.2.111 |
| Service | Zabbix Agent |

### Software

- Zabbix 7.0
- PostgreSQL
- Nginx
- Grafana 13.2.3
- Zabbix Grafana Plugin 6.8.0
- Zabbix Agent
- Amazon CloudWatch

---

## 4. Network Configuration

### VPC

```text
VPC
└── project07-vpc
    └── 10.0.0.0/16
```

### Public Subnet

```text
project07-public-subnet
└── 10.0.1.0/24
    └── project07-zabbix-server
```

Public Route Table:

```text
0.0.0.0/0
    ↓
Internet Gateway
```

### Private Subnet

```text
project07-private-subnet
└── 10.0.2.0/24
    └── project07-monitoring-target
```

Private Route Table:

```text
0.0.0.0/0
    ↓
NAT Gateway
    ↓
Internet Gateway
```

NAT Gateway는 Private Subnet의 인스턴스가 외부로 통신할 수 있도록 구성하였습니다.

> 실습 종료 후에는 NAT Gateway를 삭제하여 불필요한 비용이 발생하지 않도록 합니다.

---

## 5. Security Group

### project07-zabbix-sg

Zabbix Server에 적용되는 Security Group입니다.

허용 포트:

```text
TCP 22      SSH
TCP 8080    Zabbix Web
TCP 3000    Grafana
```

SSH 및 Web 접근은 필요한 환경에서만 허용하도록 구성합니다.

### project07-agent-sg

Monitoring Target에 적용되는 Security Group입니다.

```text
TCP 22
Source: project07-zabbix-sg

TCP 10050
Source: project07-zabbix-sg
```

Monitoring Target은 외부에서 직접 Zabbix Agent에 접근하지 못하도록 하고, Zabbix Server Security Group에서의 접근만 허용합니다.

---

## 6. Zabbix Server Installation

### Repository

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb

sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb

sudo apt update
```

### Package Installation

```bash
sudo apt install -y \
zabbix-server-pgsql \
zabbix-frontend-php \
zabbix-nginx-conf \
zabbix-sql-scripts \
zabbix-agent
```

### PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
```

Zabbix용 PostgreSQL Database를 생성합니다.

```text
Database : zabbix
User     : zabbix
```

### Database Schema Import

```bash
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```

### Zabbix Server Configuration

```bash
sudo vi /etc/zabbix/zabbix_server.conf
```

주요 설정:

```text
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=<DB_PASSWORD>
```

서비스 확인:

```bash
sudo systemctl status zabbix-server
sudo systemctl status nginx
sudo systemctl status postgresql
```

---

## 7. Zabbix Agent Configuration

Monitoring Target:

```text
10.0.2.111
```

Zabbix Agent 설정:

```bash
sudo vi /etc/zabbix/zabbix_agentd.conf
```

주요 설정:

```text
Server=10.0.1.194
ServerActive=10.0.1.194
Hostname=project07-monitoring-target
```

설정 검증:

```bash
zabbix_agentd -T -c /etc/zabbix/zabbix_agentd.conf
```

정상 결과:

```text
Validation successful
```

서비스 재시작:

```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
```

---

## 8. Zabbix Agent Connectivity Test

Zabbix Server에서 Monitoring Target의 Agent 상태를 확인합니다.

```bash
zabbix_get -s 10.0.2.111 -k agent.ping
```

정상 결과:

```text
1
```

CPU 확인:

```bash
zabbix_get -s 10.0.2.111 -k system.cpu.util
```

Zabbix Server → Monitoring Target 간 Agent 통신이 정상적으로 동작하는 것을 확인합니다.

---

## 9. Grafana Installation

Grafana를 설치하고 서비스를 활성화합니다.

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

상태 확인:

```bash
sudo systemctl status grafana-server
```

Grafana 기본 포트:

```text
TCP 3000
```

---

## 10. Zabbix Grafana Plugin

Zabbix Plugin 설치:

```bash
sudo grafana cli plugins install alexanderzobnin-zabbix-app
```

Plugin 버전:

```text
6.8.0
```

Plugin 경로:

```text
/var/lib/grafana/plugins/alexanderzobnin-zabbix-app
```

권한 설정:

```bash
sudo chown -R grafana:grafana \
/var/lib/grafana/plugins/alexanderzobnin-zabbix-app
```

Grafana 재시작:

```bash
sudo systemctl restart grafana-server
```

Grafana에서:

```text
Administration
 → Plugins and data
 → Plugins
 → Zabbix
 → Enable
```

---

## 11. Grafana - Zabbix Data Source

Grafana에서 Zabbix Data Source를 추가합니다.

```text
Data Sources
 → Add new data source
 → Zabbix
```

Zabbix API URL:

```text
http://127.0.0.1:8080/api_jsonrpc.php
```

Authentication:

```text
Auth type: User and password
Username: <ZABBIX_USERNAME>
Password: <ZABBIX_PASSWORD>
```

Save & Test를 실행하여 Zabbix Data Source 연결 상태를 확인합니다.

---

## 12. Grafana - CloudWatch Data Source

Grafana에 CloudWatch Data Source를 추가합니다.

```text
Data Sources
 → Add new data source
 → CloudWatch
```

CloudWatch를 이용하여 AWS EC2 Metric을 조회합니다.

사용 Metric:

```text
Namespace:
AWS/EC2

Metric:
CPUUtilization

Statistic:
Average

Dimension:
InstanceId

Period:
300 seconds
```

EC2 Instance:

```text
i-0eaa0f2f98ef14ae7
```

---

## 13. Monitoring Dashboard

Grafana Dashboard:

```text
Project 07 - AWS Infrastructure Monitoring
```

### Zabbix Monitoring

Zabbix를 통해 Linux 서버의 주요 시스템 리소스를 모니터링합니다.

```text
CPU Usage
Memory Usage
Disk Usage
Network Traffic
System Uptime
```

### CloudWatch Monitoring

AWS CloudWatch를 통해 EC2의 AWS Metric을 모니터링합니다.

```text
CloudWatch - EC2 CPU Utilization
```

---

## 14. Unified Monitoring

본 프로젝트에서는 Zabbix와 CloudWatch를 Grafana 하나의 Dashboard에서 통합하여 확인합니다.

```text
                    ┌──────────────┐
                    │ Zabbix Agent │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Zabbix    │
                    └──────┬───────┘
                           │
                           │
                           ▼
                    ┌──────────────┐
                    │    Grafana   │
                    └──────────────┘
                           ▲
                           │
                    ┌──────┴───────┐
                    │  CloudWatch  │
                    └──────▲───────┘
                           │
                    ┌──────┴───────┐
                    │   AWS EC2    │
                    └──────────────┘
```

### Zabbix

Linux 서버 내부의 OS 및 시스템 리소스를 수집합니다.

```text
CPU
Memory
Disk
Network
Uptime
```

### CloudWatch

AWS에서 제공하는 EC2 Metric을 수집합니다.

```text
EC2 CPUUtilization
```

### Grafana

서로 다른 Monitoring Source의 데이터를 하나의 Dashboard에서 시각화합니다.

---

## 15. AWS Systems Manager

Zabbix Server 관리 편의를 위해 AWS Systems Manager Session Manager를 구성하였습니다.

IAM Role:

```text
Project07-SSM-Role
```

Policy:

```text
AmazonSSMManagedInstanceCore
```

Session Manager를 이용하면 SSH Port를 외부에 직접 노출하지 않고 AWS Console을 통해 EC2에 접속할 수 있습니다.

---

## 16. Monitoring Verification

### Zabbix Agent

```bash
zabbix_get -s 10.0.2.111 -k agent.ping
```

Expected:

```text
1
```

### CPU

```bash
zabbix_get -s 10.0.2.111 -k system.cpu.util
```

### Zabbix Service

```bash
sudo systemctl status zabbix-server
```

### Grafana

```bash
sudo systemctl status grafana-server
```

### Nginx

```bash
sudo systemctl status nginx
```

### PostgreSQL

```bash
sudo systemctl status postgresql
```

---

## 17. Dashboard Result

### Zabbix

```text
CPU Usage
Memory Usage
Disk Usage
Network Traffic
System Uptime
```

### CloudWatch

```text
EC2 CPU Utilization
```

### Integrated Dashboard

```text
Project 07 - AWS Infrastructure Monitoring
```

Zabbix와 CloudWatch에서 수집한 Metric을 Grafana에서 하나의 Dashboard로 통합하여 AWS Infrastructure 상태를 확인할 수 있도록 구성하였습니다.

---

## 18. Troubleshooting

### Zabbix Agent Connection Rejected

증상:

```text
connection rejected
allowed hosts: "127.0.0.1"
```

확인:

```bash
sudo vi /etc/zabbix/zabbix_agentd.conf
```

설정:

```text
Server=10.0.1.194
ServerActive=10.0.1.194
```

재시작:

```bash
sudo systemctl restart zabbix-agent
```

검증:

```bash
zabbix_get -s 10.0.2.111 -k agent.ping
```

---

### Grafana Zabbix API 404

잘못된 API URL:

```text
http://127.0.0.1/api_jsonrpc.php
```

정상 API URL:

```text
http://127.0.0.1:8080/api_jsonrpc.php
```

확인:

```bash
curl -i http://127.0.0.1:8080/api_jsonrpc.php
```

HTTP POST API Endpoint가 정상적으로 존재하는지 확인합니다.

---

### Grafana CloudWatch No Data

확인 항목:

```text
Time Range
Metric
Instance ID
Period
Region
```

본 프로젝트에서는 다음과 같이 설정하여 EC2 CPU Metric을 확인하였습니다.

```text
Namespace : AWS/EC2
Metric    : CPUUtilization
Statistic : Average
Dimension : InstanceId
Period    : 300 seconds
Time Range: Last 1 hour
```

---

## 19. Project Structure

```text
PROJECT-07/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── aws-network.md
│   ├── zabbix-installation.md
│   ├── grafana-installation.md
│   ├── cloudwatch.md
│   └── troubleshooting.md
├── screenshots/
│   ├── aws-vpc.png
│   ├── aws-subnet.png
│   ├── zabbix-host.png
│   ├── grafana-zabbix.png
│   ├── grafana-cloudwatch.png
│   └── grafana-dashboard.png
└── scripts/
    └── README.md
```

---

## 20. Key Technologies

```text
AWS
├── VPC
├── EC2
├── Subnet
├── Route Table
├── Internet Gateway
├── NAT Gateway
├── Security Group
├── IAM
└── Systems Manager

Monitoring
├── Zabbix
├── Zabbix Agent
├── Amazon CloudWatch
└── Grafana

Infrastructure
├── Ubuntu Server
├── Nginx
└── PostgreSQL
```

---

## 21. Project Outcome

본 프로젝트를 통해 AWS 환경에서 다음과 같은 Infrastructure Monitoring 환경을 구축하였습니다.

- AWS VPC 기반 Public / Private Network 구성
- Public Subnet에 Zabbix Server 구축
- Private Subnet에 Monitoring Target 구축
- Zabbix Agent 기반 Linux 서버 모니터링
- AWS CloudWatch 기반 EC2 Metric 모니터링
- Grafana와 Zabbix 연동
- Grafana와 CloudWatch 연동
- Zabbix + CloudWatch 통합 Dashboard 구성
- Security Group 기반 접근 제어
- AWS Systems Manager Session Manager 구성
- Monitoring 장애 상황에 대한 Troubleshooting 수행

---

## 22. Cleanup

AWS 실습 종료 후 비용 발생을 방지하기 위해 생성한 리소스를 확인합니다.

삭제 대상 예시:

```text
EC2
NAT Gateway
Elastic IP
Load Balancer (사용 시)
EBS Volume
VPC
Subnet
Route Table
Internet Gateway
Security Group
IAM Role (불필요한 경우)
```

특히 **NAT Gateway는 실행 시간에 따라 비용이 발생할 수 있으므로 실습 종료 후 반드시 확인합니다.**

---

## 23. Next Improvement

향후 다음 기능을 추가하여 Monitoring / DevOps 프로젝트로 확장할 수 있습니다.

```text
1. Zabbix Alert / Trigger 구성
2. Slack / Email Notification
3. Python + Zabbix API 자동화
4. CloudWatch Alarm 연계
5. Grafana Alerting
6. Terraform을 이용한 AWS Infrastructure 코드화
7. Ansible을 이용한 Zabbix Agent 자동 설치
8. AWS Infrastructure as Code
```

---

## 24. Skills Demonstrated

```text
AWS Infrastructure
Linux Server
Zabbix
CloudWatch
Grafana
PostgreSQL
Nginx
VPC
Subnet
Routing
Security Group
IAM
Systems Manager
Infrastructure Monitoring
Troubleshooting
```

