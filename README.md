# Postgres_Patroni_Etcd_Cluster_Setup

# PostgreSQL Cluster with Patroni, etcd, and HAProxy

## Prerequisites
Ensure your system is up to date:
```sh
sudo apt update -y 
```

### Add PostgreSQL Repository
```sh
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ jammy-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

### Import PostgreSQL Signing Key
```sh
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

### Install PostgreSQL and Required Packages
```sh
sudo apt update 
sudo apt install -y postgresql-16 postgresql-server-dev-16
psql --version
```

### Set Hostname (Modify for Each Node)
```sh
sudo hostnamectl set-hostname nodeN  # Replace nodeN with node1, node2, or node3
```

## Install Dependencies
```sh
sudo systemctl stop postgresql
sudo apt install -y net-tools python3-pip python3-testresources
sudo pip3 install --upgrade setuptools
pip install psycopg2-binary
sudo apt install -y patroni haproxy etcd
```

## Configure etcd
Edit `/etc/default/etcd`:
```sh
ETCD_NAME="etcd3"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.31.1.161:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.31.1.161:2379"
ETCD_INITIAL_CLUSTER="etcd1=http://172.31.3.175:2380,etcd2=http://172.31.15.104:2380,etcd3=http://172.31.1.161:2380"
ETCD_INITIAL_CLUSTER_TOKEN="pg_cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
Restart etcd:
```sh
sudo systemctl restart etcd
```

## Configure Patroni
### Create Data Directory
```sh
sudo mkdir -p /data/patroni
sudo chmod 700 /data/patroni
sudo chown postgres:postgres /data/patroni
```

### Create Patroni Systemd Service
Edit `/etc/systemd/system/patroni.service`:
```ini
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
```

### Configure Patroni Cluster
Edit `/etc/patroni.yml`:
```yaml
scope: postgres
namespace: /db/
name: node3

restapi:
  listen: 172.31.1.161:8008
  connect_address: 172.31.1.161:8008

etcd:
  host: 172.31.1.161:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
  initdb:
    - encoding: UTF8
    - data-checksums

pg_hba:
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 172.31.15.104/0 md5
  - host replication replicator 172.31.3.175/0 md5
  - host replication replicator 172.31.1.161/0 md5
  - host all all 0.0.0.0/0 md5

users:
  admin:
    password: admin
    options:
      - createrole
      - createdb

postgresql:
  bin_dir: /usr/lib/postgresql/16/bin
  listen: 172.31.1.161:5432
  connect_address: 172.31.1.161:5432
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: Replicator$123
    superuser:
      username: postgres
      password: Postgres$123
  parameters:
    unix_socket_directories: '.'

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

## Restart Services
```sh
export PATH=$PATH:/usr/lib/postgresql/16/bin
sudo systemctl daemon-reload
sudo systemctl restart etcd
sudo systemctl restart patroni
sudo systemctl status patroni
```

## Configure HAProxy
Edit `/etc/haproxy/haproxy.cfg`:
```ini
global
  log /dev/log local0
  log /dev/log local1 notice
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
  log global
  mode tcp
  retries 2
  timeout client 30m
  timeout connect 4s
  timeout server 30m
  timeout check 5s

frontend postgresql_write
  bind *:5000
  mode tcp
  default_backend postgresql_primary

backend postgresql_primary
  mode tcp
  option httpchk GET /patroni
  http-check expect string "\"role\": \"primary\""
  balance roundrobin
  default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
  server node1 172.31.3.175:5432 check port 8008
  server node2 172.31.15.104:5432 check port 8008
  server node3 172.31.1.161:5432 check port 8008

frontend postgresql_read
  bind *:5001
  mode tcp
  default_backend postgresql_replicas

backend postgresql_replicas
  mode tcp
  balance roundrobin
  option tcp-check
  default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
  server node1 172.31.3.175:5432 check port 8008 backup
  server node2 172.31.15.104:5432 check port 8008
  server node3 172.31.1.161:5432 check port 8008

listen stats
  mode http
  bind *:7000
  stats enable
  stats uri /
```

## Start HAProxy
```sh
sudo systemctl restart haproxy
```

## Conclusion
Your PostgreSQL cluster is now set up with high availability using Patroni, etcd, and HAProxy. You can monitor HAProxy stats at `http://<haproxy-host>:7000/`. Happy clustering! 🚀

Any issues you can reachout to me : md.jafar95@gmail.com
