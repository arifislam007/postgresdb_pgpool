### **Methode of Procedure (MOP) for Configuring Pgpool-II on 3 PostgreSQL Servers**  
Now that you have installed **Pgpool-II** on all three PostgreSQL servers (**192.168.2.230, 192.168.2.231, 192.168.2.232**) and set up **PostgreSQL Streaming Replication**, follow these steps to configure **Pgpool-II with High Availability (HA) and Load Balancing**.  

---

## **📌 Step 1: Common Configuration for All 3 Servers**  
Edit the **Pgpool-II configuration file** on **each server**.

```bash
sudo nano /etc/pgpool-II/pgpool.conf
```

### **1️⃣ Enable Load Balancing & Streaming Replication Mode**
```ini
backend_clustering_mode = 'streaming_replication'
load_balance_mode = on
```

---

### **2️⃣ Configure Backend Nodes (PostgreSQL Servers)**
Define all **PostgreSQL database nodes**:

```ini
backend_hostname0 = '192.168.2.230'
backend_port0 = 5432
backend_weight0 = 1
backend_flag0 = 'ALWAYS_PRIMARY'

backend_hostname1 = '192.168.2.231'
backend_port1 = 5432
backend_weight1 = 1
backend_flag1 = 'ALWAYS_PRIMARY'

backend_hostname2 = '192.168.2.232'
backend_port2 = 5432
backend_weight2 = 1
backend_flag2 = 'ALWAYS_PRIMARY'
```

> 🔹 **`ALWAYS_PRIMARY`** ensures the system always treats the nodes as part of a PostgreSQL **streaming replication setup**.

---

### **3️⃣ Configure Connection Pooling**
```ini
enable_pool_hba = on
num_init_children = 32
max_pool = 4
```

---

### **4️⃣ Configure Failover & Recovery**
Define failover and automatic recovery:

```ini
failover_command = '/etc/pgpool-II/failover.sh %d %h %p'
recovery_1st_stage_command = '/etc/pgpool-II/recovery.sh %d %h %p'
```

Make the failover script executable:
```bash
sudo chmod +x /etc/pgpool-II/failover.sh
sudo chmod +x /etc/pgpool-II/recovery.sh
```

---

### **5️⃣ Configure Authentication**
Enable `pool_hba.conf` for security:

```ini
enable_pool_hba = on
```

Edit **pool_hba.conf** to allow client connections:

```bash
sudo nano /etc/pgpool-II/pool_hba.conf
```

Add:
```
host    all     all     192.168.2.0/24   md5
```

Restart Pgpool-II:
```bash
sudo systemctl restart pgpool-II
```

---

## **📌 Step 2: Configure Watchdog for High Availability**  
### **1️⃣ Create the `pgpool_node_id` File (Each Server Gets a Unique ID)**
Run this **on each server**, changing the ID:

#### On **192.168.2.230** (First Node)
```bash
echo "0" | sudo tee /etc/pgpool-II/pgpool_node_id
```

#### On **192.168.2.231** (Second Node)
```bash
echo "1" | sudo tee /etc/pgpool-II/pgpool_node_id
```

#### On **192.168.2.232** (Third Node)
```bash
echo "2" | sudo tee /etc/pgpool-II/pgpool_node_id
```

---

### **2️⃣ Enable Watchdog**
Edit **pgpool.conf** on each server:

```ini
use_watchdog = on
wd_hostname = '192.168.2.230'  # Change per server
wd_port = 9000
pgpool_node_id = 0  # Change per server
delegate_IP = '192.168.2.240'  # Virtual IP (Keep the same on all servers)
```

---

### **3️⃣ Configure Heartbeat & Peer Nodes**
Add all servers:

```ini
wd_heartbeat_port = 9694
wd_heartbeat_keepalive = 2
wd_heartbeat_deadtime = 30

heartbeat_destination0 = '192.168.2.230'
heartbeat_destination1 = '192.168.2.231'
heartbeat_destination2 = '192.168.2.232'
```

---

### **4️⃣ Restart Pgpool-II**
Run on **all three servers**:

```bash
sudo systemctl restart pgpool-II
```

---

## **📌 Step 3: Verify Pgpool-II Status**
1️⃣ **Check if all PostgreSQL nodes are connected to Pgpool-II**  
```bash
psql -p 9999 -U postgres -c "SHOW pool_nodes;"
```
Expected output:
```
 node_id |   hostname   | port  | status | lb_weight | role
---------+-------------+-------+--------+-----------+-------
 0       | 192.168.2.230 | 5432  | up     | 1.0       | primary
 1       | 192.168.2.231 | 5432  | up     | 1.0       | standby
 2       | 192.168.2.232 | 5432  | up     | 1.0       | standby
```

2️⃣ **Check Virtual IP (VIP) is Assigned**  
```bash
ip addr show
```
Ensure the **VIP (192.168.2.240)** is active.

3️⃣ **Check Watchdog Status**  
Run on **any server**:
```bash
pgpool -m fast stop
pgpool -D
```

4️⃣ **Test Load Balancing**
Run this command multiple times:
```bash
psql -h 192.168.2.240 -U postgres -d mydb -p 9999 -c "SELECT inet_server_addr();"
```
It should return different node IPs, meaning load balancing is working.

---

## **📌 Step 4: Test Automatic Failover**
1️⃣ **Stop PostgreSQL on the Master**
```bash
sudo systemctl stop postgresql
```
2️⃣ **Monitor Failover Logs**
```bash
journalctl -u pgpool-II -f
```
3️⃣ **Verify New Master**
```bash
psql -p 9999 -U postgres -c "SHOW pool_nodes;"
```
4️⃣ **Restart the Old Master**
```bash
sudo systemctl start postgresql
```

---

## **🎯 Summary**
✅ Installed and configured **Pgpool-II** on 3 PostgreSQL nodes.  
✅ Configured **Streaming Replication & Load Balancing**.  
✅ Enabled **Watchdog for HA**.  
✅ Configured **Failover & Virtual IP**.  
✅ Verified **Pgpool-II operation & failover**.  

🚀 **You are now running a fully HA PostgreSQL Cluster with Pgpool-II!** 🚀
