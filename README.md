# Containerized-Zabbix-6.4-Stack-with-Multi-Node-Monitoring

This project demonstrates the deployment of a full-stack Zabbix 6.4 monitoring system on RHEL 9 using rootless Podman Pods, with successful cross-VM monitoring of external Rocky Linux nodes.

The goal is not only deployment, but also solving real-world issues related to:

- Container networking
- Cross-host communication
- SELinux & firewall restrictions
- 
## 🏗️ Architecture

Host Environment
- RHEL 9 (Rootless Podman)

Container Orchestration
- Podman Pod (shared network namespace)

Core Components
- Zabbix-DB: MySQL 8.0 (with persistent volume)
- Zabbix-Server: Alpine-based Zabbix Server 6.4
- Zabbix-Web: Apache-based frontend (port 8080)

Monitored Node
- External Rocky Linux VM (running Zabbix Agent 2)
