# 🚧 Troubleshooting & Key Learnings

## ⚠️ Issue #1: Missing Environment File
```bash
Error: parsing file "./zabbix.env": no such file or directory
```
Explanation:
The container relies on environment variables (DB name, user, password).
Without zabbix.env, MySQL cannot initialize.


## ⚠️ Issue #2: Can't connect to local server through socket
```bash
 Can't connect to local server through socket '/run/mysqld/mysqld.sock' (2)

     2:20260401:083946.358 database is down: reconnecting in 10 seconds

     2:20260401:083956.360 [Z3001] connection to database 'zabbix' failed: [2002] Can't connect to local server through socket '/run/mysqld/mysqld.sock' (2)

     2:20260401:083956.360 database is down: reconnecting in 10 seconds
```

## ⚠️ Issue #3: Database error
<img width="1247" height="380" alt="Screenshot 2026-04-01 164921" src="https://github.com/user-attachments/assets/0de1810b-67f2-43bc-803c-00602322fae8" />
### 🧠 Detailed Explanation

In a traditional Linux architecture, when a database client (such as PHP or the Zabbix Server backend written in C) detects that the database host is set to `localhost`, it triggers an optimization mechanism:

➡️ Instead of using the network stack (TCP/IP), it directly attempts to communicate via a **Unix domain socket** (`.sock` file) on the local filesystem.
---
### 🚧 Problem in Podman Pod Environment
- **Isolation**  
  The `zabbix-db` container does create the `.sock` file, but it exists only inside its **own container filesystem**.

- **Missing Mapping**  
  The `zabbix-server` container does **not** have access to that socket file.

- **Key Contradiction**  
  Even though containers are in the same Pod (sharing the same IP), they **do NOT share filesystem paths like `/run`**.
---
### ✅ Solution Logic
Force the use of TCP/IP instead of Unix socket:
<img width="1262" height="570" alt="image" src="https://github.com/user-attachments/assets/bf791b06-0a96-4825-ad78-3c019b4b60d3" />

## ⚠️ Issue #4: Host Connection Failure (Connection refused)
<img width="1737" height="199" alt="Screenshot 2026-04-01 171220" src="https://github.com/user-attachments/assets/e2099c7b-2752-4a8a-b90f-bbbf9a48bf73" />

```bash
zabbix_get: [111] Connection refused
```
### 🔎 Troubleshooting Workflow (Step-by-Step)
For cross-host (RHEL → Rocky) or container → host scenarios

1️⃣ Process Listening Check
```bash
ss -tulpn | grep 10050
```
Key Point:
```bash
0.0.0.0:10050  or  *:10050
```
❌ If you see:
```bash
127.0.0.1:10050
```
→ The agent only accepts local connections, external requests will be rejected.

2️⃣ Zabbix Agent Access Control

Check config:
```bash
zabbix_agent2.conf

Key Parameters:

Server=
ServerActive=
```
Important:
- Zabbix Agent has built-in access control
- If source IP is NOT in Server, it will immediately drop the connection

Configuration Rules:
- Cross-host scenario → use RHEL physical IP
- Rootless Podman (same host) → may need:
10.0.2.2
  
3️⃣ Firewall Check
```bash
sudo firewall-cmd --list-all
```
Key Point:
- Ensure 10050/tcp is allowed

Even if the process is listening:
👉 Firewalld can still reject packets before they reach the process

Try
```bash
sudo systemctl stop firewalld
```
If issue disappears → firewall rules are misconfigured

4️⃣ SELinux Check
Key Point:
- RHEL 9 enables SELinux by default
It may block:
- Outbound connections (Zabbix Server → Agent)
- Non-standard port communication

Try
```bash
sudo setenforce 0
```

5️⃣ Network Routing & Virtualization
```bash
Test connectivity from inside container:
nc -zv <IP> 10050
```
Zabbix_get with correct host & client IP
```bash
[danny@rhel ~]$ podman exec -it zabbix-server sh
~ $ zabbix_get -s 10.0.2.2 -p 10050 -k system.hostname
zabbix_get [254]: Get value error: cannot connect to [[10.0.2.2]:10050]: [111] Connection refused
~ $ zabbix_get -s 192.168.112.139 -p 10050 -k system.hostname
zabbix_get [255]: Get value error: cannot connect to [[192.168.112.139]:10050]: [111] Connection refused
~ $ zabbix_get -s 192.168.112.136 -p 10050 -k system.hostname
rocky
```




