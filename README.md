# PostgreSQL 18 HA PoC using Docker Compose inside WSL2

Simple PostgreSQL 18 High Availability (HA) Proof of Concept using:

- PostgreSQL 18
- Docker Compose
- WSL2 Ubuntu
- Streaming Replication
- Manual Failover

This PoC is designed for learning PostgreSQL replication concepts without:

- Patroni
- Kubernetes
- Docker Swarm
- Docker Desktop
- Cloud Infrastructure

Focus area of this PoC:

- Docker container behavior during replication
- PostgreSQL WAL streaming
- Manual failover concepts
- Standby promotion
- Multi-standby architecture

---

# Architecture

```text
                Windows 11
                     |
                 WSL2 Ubuntu
                     |
              Docker Compose
                     |
        -------------------------------------------------
        |               |               |               |
    pg-primary      pg-standby1     pg-standby2     pg-standby3

Streaming Replication:

pg-primary  --->  pg-standby1
pg-primary  --->  pg-standby2
pg-primary  --->  pg-standby3
```

---

# Environment

| Component | Version |
|---|---|
| Windows | Windows 11 |
| WSL | WSL2 Ubuntu |
| Docker | Docker Engine |
| PostgreSQL | PostgreSQL 18 |
| Replication | Streaming Replication |

---

# STEP 1 — Install Docker Compose

Install docker-compose:

```bash
sudo apt update

sudo apt install -y docker-compose
```

Verify:

```bash
docker-compose --version
```

Expected:

```text
docker-compose version 1.29.x
```

---

# STEP 2 — Create Project Directory

```bash
mkdir ~/pg-ha-compose

cd ~/pg-ha-compose
```

---

# STEP 3 — Create docker-compose.yml

Create file:

```bash
vi docker-compose.yml
```

Paste:

```yaml
version: '3.9'

services:

  pg-primary:
    image: postgres:18
    container_name: pg-primary
    hostname: pg-primary
    ports:
      - "5434:5432"
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./primary:/var/lib/postgresql/18/docker
    networks:
      - pgnet

  pg-standby1:
    image: postgres:18
    container_name: pg-standby1
    hostname: pg-standby1
    ports:
      - "6001:5432"
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./standby1:/var/lib/postgresql/18/docker
    networks:
      - pgnet

  pg-standby2:
    image: postgres:18
    container_name: pg-standby2
    hostname: pg-standby2
    ports:
      - "6002:5432"
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./standby2:/var/lib/postgresql/18/docker
    networks:
      - pgnet

  pg-standby3:
    image: postgres:18
    container_name: pg-standby3
    hostname: pg-standby3
    ports:
      - "6003:5432"
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./standby3:/var/lib/postgresql/18/docker
    networks:
      - pgnet

networks:
  pgnet:
```

Save:

```text
ESC
:wq
```

---

# STEP 4 — Start Containers

```bash
docker-compose up -d
```

Verify:

```bash
docker ps
```

Expected:

```text
pg-primary
pg-standby1
pg-standby2
pg-standby3
```

---

# STEP 5 — Configure PRIMARY

Connect:

```bash
docker exec -it pg-primary psql -U postgres
```

Create replication user:

```sql
CREATE ROLE replicator
WITH REPLICATION
LOGIN
PASSWORD 'replpass';
```

Verify replication settings:

```sql
SHOW wal_level;

SHOW max_wal_senders;
```

Expected:

```text
replica
10
```

Exit:

```sql
\q
```

---

# STEP 6 — Install vim Inside Container

Enter container:

```bash
docker exec -it pg-primary bash
```

Install vim:

```bash
apt update

apt install -y vim
```

---

# STEP 7 — Configure pg_hba.conf

Find data directory:

```bash
psql -U postgres -c "SHOW data_directory;"
```

Expected:

```text
/var/lib/postgresql/18/docker
```

Edit pg_hba.conf:

```bash
vi /var/lib/postgresql/18/docker/pg_hba.conf
```

Add at END:

```text
host replication replicator 0.0.0.0/0 scram-sha-256
```

Save:

```text
ESC
:wq
```

Exit container:

```bash
exit
```

---

# STEP 8 — Restart PRIMARY

```bash
docker restart pg-primary
```

---

# STEP 9 — Create Base Backup

Enter PRIMARY container:

```bash
docker exec -it pg-primary bash
```

Create backup directory:

```bash
mkdir /tmp/standby
```

Run pg_basebackup:

```bash
/usr/lib/postgresql/18/bin/pg_basebackup \
-h pg-primary \
-D /tmp/standby \
-U replicator \
-P \
-W \
-R
```

Password:

```text
replpass
```

Expected:

```text
base backup completed
```

Exit:

```bash
exit
```

---

# STEP 10 — Copy Backup To WSL Host

```bash
docker cp pg-primary:/tmp/standby ./standby_base
```

Verify:

```bash
ls -ltr standby_base/standby
```

You should see:

```text
PG_VERSION
base
global
pg_wal
standby.signal
```

---

# STEP 11 — Stop Containers

```bash
docker-compose down
```

---

# STEP 12 — Remove Old Standby Directories

```bash
sudo rm -rf standby1
sudo rm -rf standby2
sudo rm -rf standby3
```

---

# STEP 13 — Create Fresh Standby Directories

```bash
mkdir standby1
mkdir standby2
mkdir standby3
```

---

# STEP 14 — Copy Base Backup To All Standbys

```bash
cp -r standby_base/standby/* standby1/

cp -r standby_base/standby/* standby2/

cp -r standby_base/standby/* standby3/
```

---

# STEP 15 — Fix Ownership

```bash
sudo chown -R 999:999 standby1
sudo chown -R 999:999 standby2
sudo chown -R 999:999 standby3
```

---

# STEP 16 — Start Environment Again

```bash
docker-compose up -d
```

Verify:

```bash
docker ps
```

Expected:

```text
pg-primary
pg-standby1
pg-standby2
pg-standby3
```

---

# STEP 17 — Verify Replication

Connect PRIMARY:

```bash
docker exec -it pg-primary psql -U postgres
```

Run:

```sql
SELECT application_name,
       client_addr,
       state
FROM pg_stat_replication;
```

Expected:

```text
streaming
streaming
streaming
```

---

# STEP 18 — Test Replication

On PRIMARY:

```sql
CREATE TABLE test1(id int, name text);

INSERT INTO test1 VALUES (1,'Clement');
```

Exit:

```sql
\q
```

---

# STEP 19 — Verify Data On Standby

Connect standby:

```bash
docker exec -it pg-standby1 psql -U postgres
```

Run:

```sql
SELECT * FROM test1;
```

Expected:

```text
1 | Clement
```

SUCCESS.

---

# STEP 20 — Verify Read-Only Standby

Run on standby:

```sql
INSERT INTO test1 VALUES (2,'TEST');
```

Expected:

```text
cannot execute INSERT in a read-only transaction
```

GOOD.

---

# STEP 21 — Manual Failover

Stop PRIMARY:

```bash
docker stop pg-primary
```

---

# STEP 22 — Promote Standby

Enter standby:

```bash
docker exec -it pg-standby1 bash
```

Switch user:

```bash
su - postgres
```

Promote:

```bash
/usr/lib/postgresql/18/bin/pg_ctl promote \
-D /var/lib/postgresql/18/docker
```

Expected:

```text
server promoting
```

Exit:

```bash
exit
```

---

# STEP 23 — Verify New PRIMARY

Connect standby1:

```bash
docker exec -it pg-standby1 psql -U postgres
```

Run:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

Meaning:

```text
NOW PRIMARY
```

---

# STEP 24 — Test Writes After Failover

Run:

```sql
INSERT INTO test1 VALUES (2,'Failover Success');

SELECT * FROM test1;
```

SUCCESS.

---

# Important Concepts Learned

| Concept | Description |
|---|---|
| Docker Compose | Multi-container orchestration |
| Streaming Replication | WAL-based replication |
| WAL Sender | Primary sends WAL |
| WAL Receiver | Standby receives WAL |
| pg_basebackup | Physical standby creation |
| standby.signal | Enables standby mode |
| pg_stat_replication | Monitor replication |
| pg_ctl promote | Promote standby |
| pg_is_in_recovery() | Identify primary/standby |
| Manual Failover | Promote standby manually |

---

# Final Learning Outcome

This PoC demonstrates:

- PostgreSQL streaming replication
- Multi-standby architecture
- Docker container behavior
- WAL shipping
- Read-only standby
- Manual failover
- Standby promotion
- Docker Compose orchestration
- PostgreSQL 18 container storage behavior

without using:

- Patroni
- Kubernetes
- Docker Swarm
- Cloud HA tools

# Cleanup Complete PoC

Stop and remove all containers:

```bash
docker-compose down
```

Verify:

```bash
docker ps
```

Remove PostgreSQL Docker images (optional):

```bash
docker rmi postgres:18
```

Remove project folders:

```bash
cd ~

sudo rm -rf ~/pg-ha-compose
```

Remove unused Docker resources:

```bash
docker system prune -a
```

Remove unused Docker volumes:

```bash
docker volume prune
```

Verify cleanup:

```bash
docker ps -a

docker images

docker volume ls
```

Expected:

```text
No PostgreSQL containers
No PostgreSQL images
No unused volumes
```
