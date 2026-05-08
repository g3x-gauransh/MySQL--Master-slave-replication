# MySQL Master-Slave Replication with Docker

A Docker-based MySQL replication setup with one master and one slave instance, useful for testing replication behavior locally.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed
- Port `3307` and `3308` free on your machine

## Project Structure

```
.
├── docker-compose.yml
├── .env
├── master.cnf
└── slave.cnf
```

## Setup

**1. Clone the repository**
```bash
git clone https://github.com/<your-username>/mysql-replication.git
cd mysql-replication
```

**2. Create your `.env` file**
```bash
cp .env.example .env
```
Open `.env` and set your password:
```env
MYSQL_ROOT_PASSWORD=your_password_here
```

**3. Start the containers**
```bash
docker compose up -d
```

**4. Verify both containers are running**
```bash
docker ps
```

## Connecting

| Instance | Host | Port |
|----------|------|------|
| Master | `127.0.0.1` | `3307` |
| Slave | `127.0.0.1` | `3308` |

Connect via any MySQL client:
```bash
mysql -h 127.0.0.1 -P 3307 -u root -p   # master
mysql -h 127.0.0.1 -P 3308 -u root -p   # slave
```

## Testing Replication

**1. Verify replication threads are running**

Connect to the slave and check status:
```bash
docker exec -it mysql-slave mysql -uroot -p$MYSQL_ROOT_PASSWORD
```
```sql
SHOW REPLICA STATUS\G
```
Both of these must be `Yes` before proceeding:
```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
```

**2. Basic replication test**

On the master, create a table and insert a row:
```sql
-- master (port 3307)
USE testdb;
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO users VALUES (1, 'test_user');
```

On the slave, verify it came through:
```sql
-- slave (port 3308)
USE testdb;
SELECT * FROM users;
```

**3. Replication lag test**

Bulk insert on the master to create observable lag:
```sql
-- master
INSERT INTO users
SELECT id + 1, CONCAT('user_', id)
FROM (
  WITH RECURSIVE seq(id) AS (
    SELECT 2 UNION ALL SELECT id+1 FROM seq WHERE id < 10000
  ) SELECT id FROM seq
) t;
```

Immediately check lag on the slave:
```sql
-- slave
SHOW REPLICA STATUS\G
-- watch Seconds_Behind_Source drop to 0 over time
```

This demonstrates eventual consistency — the slave will catch up, but reads during that window may return stale data.

**4. Confirm slave is read-only**

Writes to the slave should be rejected:
```sql
-- slave
INSERT INTO users VALUES (99999, 'readonly_test');
-- Expected: ERROR 1290
```

If it doesn't error, add `read_only = ON` to your `slave.cnf` and restart the container.

## Teardown

```bash
docker compose down
```

To also remove volumes:
```bash
docker compose down -v
```