

## 1. Project Overview

AWS 환경에서 **Zabbix + AWS CloudWatch + Grafana**를 연계하여 서버 및 AWS 인프라 모니터링 환경을 구축합니다.

- Zabbix를 이용한 Linux 서버 리소스 모니터링
- Amazon CloudWatch를 이용한 AWS EC2 모니터링
- Grafana를 이용한 통합 모니터링 대시보드 구성
- AWS VPC / Public Subnet / Private Subnet 기반의 네트워크 구성
- Security Group을 이용한 모니터링 서버와 대상 서버 간 접근 제어
- AWS Systems Manager Session Manager를 이용한 관리 환경 구성

---

## 2. Project Architecture

```text
                         AWS
                          │
                  ┌───────▼───────┐
                  │      VPC      │
                  │ 10.0.0.0/16   │
                  └───────┬───────┘
                          │
             ┌────────────┴────────────┐
             │                         │
     Public Subnet              Private Subnet
      10.0.1.0/24                10.0.2.0/24
             │                         │
             ▼                         ▼
   ┌──────────────────┐      ┌──────────────────┐
   │ Zabbix Server    │      │ Monitoring Target│
   │ Ubuntu 24.04     │      │ Ubuntu 24.04     │
   │                  │      │                  │
   │ Zabbix           │      │ Zabbix Agent     │
   │ PostgreSQL       │◄─────│                  │
   │ Nginx            │      └──────────────────┘
   │ Grafana          │
   └────────┬─────────┘
            │
            │
      ┌─────┴─────┐
      │           │
      ▼           ▼
   Zabbix      CloudWatch
      │           │
      └─────┬─────┘
            ▼
       ┌──────────┐
       │ Grafana  │
       │ Dashboard│
       └──────────┘

```

---

## 3. Project Environment
<img width="1149" height="270" alt="image" src="https://github.com/user-attachments/assets/0916dd08-ae31-4c58-8cce-019dca6ee91a" />

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
<img width="1376" height="551" alt="image" src="https://github.com/user-attachments/assets/0c3ae90d-afa5-4a94-b04a-30f5acf4226b" />

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
<img width="1537" height="137" alt="image" src="https://github.com/user-attachments/assets/93bf5998-0f63-49a0-a90e-db2beec267ca" />

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

<img width="1821" height="849" alt="image" src="https://github.com/user-attachments/assets/61c38212-02ec-44c3-95db-f20402e8fec5" />


Grafana Dashboard:

```text
Project 07 - AWS Infrastructure Monitoring
```

### Zabbix Monitoring

<img width="1852" height="896" alt="image" src="https://github.com/user-attachments/assets/f491b04d-d9e2-46c5-8ea7-85e333d24a57" />

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
<img width="543" height="82" alt="image" src="https://github.com/user-attachments/assets/aee5249b-8c0f-40fd-b9b6-cd2177368c1a" />
터미널에서 실제 연결 상태를 증명합니다.
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




---

## 19. Key Technologies

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



```

---



