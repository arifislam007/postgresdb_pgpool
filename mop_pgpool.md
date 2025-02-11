### **Pgpool-II Multi-Node Setup (Manual Failover & Read Distribution)**
You want to configure **Pgpool-II on all three PostgreSQL servers** (`192.168.2.230`, `192.168.2.231`, `192.168.2.232`) while avoiding automatic failover and recovery.  
This setup will:
✅ **Manually control writes on the master**  
✅ **Distribute read queries across all nodes**  
✅ **Use Watchdog for monitoring**  

---

## **📌 Step 1: Install Pgpool-II (Already Done)**
Ensure Pgpool-II is installed on **all three servers**:

```bash
sudo apt update && sudo apt install -y pgpool2
```

---

## **📌 Step 2: Configure Pgpool-II (`pgpool.conf`) on Each Server**
Edit the configuration file on each server:

```bash
sudo nano /etc/pgpool-II/pgpool.conf
```

### **🔹 Common Settings for All Nodes**
```ini
# Enable load balancing
load_balance_mode = on

# Disable automatic failover (manual failover setup)
failover_command = ''
failback_command = ''
follow_master_command = ''

# Enable streaming replication mode
replication_mode = on
enable_pool_hba = on
master_slave_mode = on
master_slave_sub_mode = 'stream'

# Connection Pool Settings
num_init_children = 32
max_pool = 4
child_life_time = 300
client_idle_limit = 0

# Health check settings
health_check_period = 10
health_check_timeout = 5
health_check_user = 'pgpool'
health_check_password = 'your_password'
health_check_database = 'postgres'

# Set the backend nodes (PostgreSQL servers)
backend_hostname0 = '192.168.2.230'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/data/pgdata/16/data'
backend_flag0 = 'ALWAYS_MASTER'   # This is the MASTER

backend_hostname1 = '192.168.2.231'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/data/pgdata/16/data'
backend_flag1 = 'ALWAYS_PRIMARY'  # This is a read replica

backend_hostname2 = '192.168.2.232'
backend_port2 = 5432
backend_weight2 = 1
backend_data_directory2 = '/data/pgdata/16/data'
backend_flag2 = 'ALWAYS_PRIMARY'  # This is a read replica
```

---

### **🔹 Watchdog Settings for HA**
```ini
use_watchdog = on
trusted_servers = '8.8.8.8, 8.8.4.4'

# Each node should have a unique node ID
wd_hostname = '192.168.2.230'  # Change this for each server
wd_port = 9000
pgpool_node_id = 0  # Change this for each server (0, 1, 2)

# Heartbeat settings (used for node failure detection)
wd_heartbeat_port = 9694
wd_heartbeat_keepalive = 2
wd_heartbeat_deadtime = 30

# Other Pgpool nodes
heartbeat_destination0 = '192.168.2.231'
heartbeat_destination1 = '192.168.2.232'

# Virtual IP (used for failover)
delegate_IP = '192.168.2.240'  # Floating IP for HA (Set once)
```

🔹 **On `192.168.2.231` (Node 2), change:**
```ini
wd_hostname = '192.168.2.231'
pgpool_node_id = 1
```

🔹 **On `192.168.2.232` (Node 3), change:**
```ini
wd_hostname = '192.168.2.232'
pgpool_node_id = 2
```

---

## **📌 Step 3: Configure Authentication (`pool_hba.conf`)**
Edit the `pool_hba.conf` file:

```bash
sudo nano /etc/pgpool-II/pool_hba.conf
```

Add the following lines to allow connections from trusted IPs:

```ini
# Allow local connections
local   all         all                               trust

# Allow Pgpool servers to communicate
host    all         all         192.168.2.230/32     md5
host    all         all         192.168.2.231/32     md5
host    all         all         192.168.2.232/32     md5
```

---

## **📌 Step 4: Create Pgpool Admin User in PostgreSQL**
On **all three PostgreSQL servers**, log into `psql`:

```bash
psql -U postgres
```

Create a Pgpool monitoring user:

```sql
CREATE USER pgpool WITH PASSWORD 'your_password' SUPERUSER;
```

---

## **📌 Step 5: Set Node ID for Watchdog**
On **each server**, set a unique node ID:

```bash
echo "0" | sudo tee /etc/pgpool-II/pgpool_node_id  # On 192.168.2.230
echo "1" | sudo tee /etc/pgpool-II/pgpool_node_id  # On 192.168.2.231
echo "2" | sudo tee /etc/pgpool-II/pgpool_node_id  # On 192.168.2.232
```

---

## **📌 Step 6: Restart Pgpool on All Servers**
```bash
sudo systemctl restart pgpool-II
sudo systemctl enable pgpool-II
```

Check status:
```bash
sudo systemctl status pgpool-II
```

---

## **📌 Step 7: Verify Pgpool-II**
### **Check Load Balancing**
Run:
```bash
psql -h 192.168.2.230 -U postgres -d postgres -p 9999
```
Run a read query multiple times:
```sql
SELECT inet_server_addr();
```
It should **return different node IPs** (showing load balancing is working).

### **Check Pgpool Nodes**
```bash
psql -p 9999 -U postgres -c "SHOW pool_nodes;"
```
It should list:
```ini
 node_id |  hostname   | port  | status  |  role   
---------+------------+-------+---------+---------
 0       | 192.168.2.230 | 5432  | up     | primary
 1       | 192.168.2.231 | 5432  | up     | standby
 2       | 192.168.2.232 | 5432  | up     | standby
```

---

### **📌 Step 8: Test Failover (Manual)**
Since **automatic failover is disabled**, you will need to manually promote a standby if the master fails.

1. **Stop the master (192.168.2.230)**:
   ```bash
   sudo systemctl stop postgresql
   ```

2. **Promote a standby manually**:
   Run this on one of the replicas (e.g., `192.168.2.231`):
   ```bash
   pg_ctl promote -D /data/pgdata/16/data
   ```

3. **Update `pgpool.conf` on all nodes** to mark the new master.

4. **Restart Pgpool-II** on all nodes:
   ```bash
   sudo systemctl restart pgpool-II
   ```

---

## **✅ Summary of Configuration**
| Server IP       | Role          | Pgpool Node ID | Watchdog Hostname |
|----------------|--------------|---------------|------------------|
| 192.168.2.230 | Master (Primary) | `0`           | `192.168.2.230`  |
| 192.168.2.231 | Standby (Replica) | `1`           | `192.168.2.231`  |
| 192.168.2.232 | Standby (Replica) | `2`           | `192.168.2.232`  |

---

## **🚀 Next Steps**
✅ **Test read query load balancing** using `SELECT inet_server_addr();`  
✅ **Test manual failover** by promoting a replica manually  
✅ **Monitor logs** with `journalctl -u pgpool-II -f`  

Let me know if you need **automation scripts** or **further tuning**! 🚀
