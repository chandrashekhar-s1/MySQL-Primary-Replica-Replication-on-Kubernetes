# MySQL Primary-Replica Replication on Kubernetes

This project demonstrates how to configure **MySQL Primary-Replica Replication** using Kubernetes StatefulSets, Headless Services, Persistent Volumes, and MySQL GTID-based replication.

## Architecture

```text
                    Kubernetes DNS
                         │
                         ▼
                  mysql-0.mysql
                         │
                         │ Port 3306
                         │ MySQL Replication
                         ▼
                  ┌─────────────┐
                  │   mysql-1   │
                  │   REPLICA   │
                  │  server-id=2│
                  └─────────────┘
                         ▲
                         │
                  ┌─────────────┐
                  │   mysql-0   │
                  │   PRIMARY   │
                  │  server-id=1│
                  └─────────────┘
```

### Components

* **mysql-0** — MySQL Primary
* **mysql-1** — MySQL Replica
* **Headless Service** — Provides stable DNS names
* **StatefulSet** — Provides stable Pod names
* **PersistentVolumeClaim** — Provides persistent database storage
* **ConfigMap** — Stores MySQL configuration
* **Secret** — Stores MySQL credentials
* **GTID Replication** — Synchronizes data from Primary to Replica

---

# 1. Check Pod Connectivity

First, verify that `mysql-1` can communicate with `mysql-0`.

Run the following command from the Replica:

```bash
kubectl exec -n mysql mysql-1 -- \
mysql -h mysql-0.mysql -uroot -prootpassword \
-e "SELECT @@hostname, @@server_id;"
```

Expected output:

```text
@@hostname    @@server_id
mysql-0       1
```

If this command works, `mysql-1` can connect to the MySQL server running on `mysql-0`.

The Kubernetes DNS name is:

```text
mysql-0.mysql
```

---

# 2. Create Replication User on Primary

Connect to the Primary MySQL Pod:

```bash
kubectl exec -it -n mysql mysql-0 -- \
mysql -uroot -prootpassword
```

Create a replication user:

```sql
CREATE USER 'repl'@'%'
IDENTIFIED WITH mysql_native_password BY 'replpassword';
```

Grant replication permissions:

```sql
GRANT REPLICATION SLAVE, REPLICATION CLIENT
ON *.* TO 'repl'@'%';
```

Apply the privileges:

```sql
FLUSH PRIVILEGES;
```

Verify the replication user:

```sql
SELECT user, host, plugin
FROM mysql.user
WHERE user = 'repl';
```

Expected:

```text
repl    %    mysql_native_password
```

---

# 3. Test Replication User Connectivity

From `mysql-1`, connect to the Primary using the replication user:

```bash
kubectl exec -n mysql mysql-1 -- \
mysql -h mysql-0.mysql -urepl -preplpassword \
-e "SELECT 1;"
```

Expected output:

```text
1
1
```

If this command succeeds, the Replica can authenticate to the Primary using the replication account.

---

# 4. Configure the Replica

Connect to `mysql-1`:

```bash
kubectl exec -it -n mysql mysql-1 -- \
mysql -uroot -prootpassword
```

Stop any existing replication:

```sql
STOP REPLICA;
```

Reset the existing replication configuration:

```sql
RESET REPLICA ALL;
```

Configure the Primary as the replication source:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-0.mysql',
  SOURCE_PORT=3306,
  SOURCE_USER='repl',
  SOURCE_PASSWORD='replpassword',
  SOURCE_AUTO_POSITION=1;
```

Start replication:

```sql
START REPLICA;
```

---

# 5. Check Replication Status

Run on `mysql-1`:

```sql
SHOW REPLICA STATUS\G
```

The following values should be:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

You may also see:

```text
Replica_IO_State: Waiting for source to send event
```

This is normal.

It means the Replica I/O thread is connected to the Primary and is waiting for new binary log events.

A healthy replication status looks like:

```text
Replica_IO_State: Waiting for source to send event
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

---

# 6. Test MySQL Replication

## Create Database on Primary

Connect to `mysql-0`:

```bash
kubectl exec -it -n mysql mysql-0 -- \
mysql -uroot -prootpassword
```

Create a database:

```sql
CREATE DATABASE testdb;
```

Create a table:

```sql
USE testdb;

CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);
```

Insert data:

```sql
INSERT INTO users VALUES
(1, 'Chandrashekhar'),
(2, 'DevOps Engineer');
```

---

# 7. Verify Data on Replica

Connect to `mysql-1`:

```bash
kubectl exec -n mysql mysql-1 -- \
mysql -uroot -prootpassword \
-e "SHOW DATABASES;"
```

You should see:

```text
testdb
```

Check the table:

```bash
kubectl exec -n mysql mysql-1 -- \
mysql -uroot -prootpassword \
-e "SELECT * FROM testdb.users;"
```

Expected:

```text
+----+-----------------+
| id | name            |
+----+-----------------+
|  1 | Chandrashekhar  |
|  2 | DevOps Engineer  |
+----+-----------------+
```

This confirms that data created on the Primary is replicated to the Replica.

---

# Replication Flow

```text
                MySQL Primary
                 mysql-0
               server-id=1
                    │
                    │
                    │ Binary Log
                    │
                    ▼
             mysql-0.mysql:3306
                    │
                    │
                    ▼
                MySQL Replica
                 mysql-1
               server-id=2
                    │
                    ▼
              Apply Transactions
```

---

# Important Notes

## Unique Server IDs

Each MySQL server must have a unique `server-id`.

Primary:

```ini
server-id=1
```

Replica:

```ini
server-id=2
```

Do not use the same `server-id` on both MySQL instances.

---

## Headless Service

The Kubernetes Headless Service provides stable DNS for StatefulSet Pods.

For example:

```text
mysql-0.mysql
mysql-1.mysql
```

The Replica uses:

```text
mysql-0.mysql
```

to connect to the Primary.

---

## MySQL Authentication

For this lab setup, the replication user uses:

```text
mysql_native_password
```

This avoids the `caching_sha2_password` secure-connection error when replication is configured without TLS.

For production environments, use **TLS/SSL-secured MySQL replication** instead.

---

## Existing Data

This setup assumes that `mysql-1` is a fresh or empty Replica.

If `mysql-0` and `mysql-1` already contain different data, configure an initial data synchronization before starting replication.

Otherwise, replication can fail with errors such as:

```text
Duplicate entry
```

or:

```text
Database already exists
```

The recommended flow for a new lab environment is:

```text
1. Create mysql-0 (Primary)
        ↓
2. Create mysql-1 (Replica)
        ↓
3. Configure unique server IDs
        ↓
4. Create replication user
        ↓
5. Configure GTID replication
        ↓
6. Start Replica
        ↓
7. Verify Replica_IO_Running = Yes
        ↓
8. Verify Replica_SQL_Running = Yes
        ↓
9. Create test data on mysql-0
        ↓
10. Verify data on mysql-1
```

---

# Useful Commands

Check Pods:

```bash
kubectl get pods -n mysql -o wide
```

Check Service:

```bash
kubectl get svc -n mysql
```

Check StatefulSet:

```bash
kubectl get statefulset -n mysql
```

Check PVCs:

```bash
kubectl get pvc -n mysql
```

Check MySQL logs:

```bash
kubectl logs -n mysql mysql-0
```

```bash
kubectl logs -n mysql mysql-1
```

Check replication status:

```bash
kubectl exec -it -n mysql mysql-1 -- \
mysql -uroot -prootpassword \
-e "SHOW REPLICA STATUS\G"
```

Check MySQL server IDs:

```bash
kubectl exec -n mysql mysql-0 -- \
mysql -uroot -prootpassword \
-e "SHOW VARIABLES LIKE 'server_id';"

kubectl exec -n mysql mysql-1 -- \
mysql -uroot -prootpassword \
-e "SHOW VARIABLES LIKE 'server_id';"
```

---

# Result

The final Kubernetes MySQL architecture is:

```text
                    Kubernetes
                       │
                 StatefulSet
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
      mysql-0                   mysql-1
      PRIMARY                   REPLICA
      ID: 1                     ID: 2
          │                         ▲
          │                         │
          └────── GTID Replication ─┘
                    │
                    ▼
             Data Synchronization
```

The replication is considered healthy when:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

At this point, changes made on `mysql-0` are replicated to `mysql-1`.