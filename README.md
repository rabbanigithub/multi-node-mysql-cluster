# Create a Multi-Node MySQL Cluster on Ubuntu 22.04

In this LAB will need a total of three servers: two servers for the redundant MySQL data nodes (`ndbd`), and one server for the Cluster Manager (`ndb_mgmd`) and MySQL server/client (`mysqld` and `mysql`).

- `10.11.12.41` will be the Cluster Manager & MySQL server node
- `10.11.12.42` will be the first MySQL Cluster data node
- `10.11.12.43` will be the second data node

[Official MySQL Cluster download page](https://dev.mysql.com/downloads/cluster/)

## Step 1 - Installing and Configuring the Cluster Manager

_Execute step 1 into Cluster Manager server `10.11.12.41`_

* Install `ndb_mgmd` using `dpkg`
```
wget https://cdn.mysql.com//Downloads/MySQL-Cluster-8.0/mysql-cluster-community-management-server_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-cluster-community-management-server_8.0.31-1ubuntu22.04_amd64.deb
```
* Configure `ndb_mgmd`
```
sudo mkdir /var/lib/mysql-cluster
sudo vi /var/lib/mysql-cluster/config.ini
```
Paste the following text into editor
```
NoOfReplicas=2

[ndb_mgmd]
hostname=10.11.12.41 # Hostname of the manager
datadir=/var/lib/mysql-cluster  # Directory for the log files

[ndbd]
hostname=10.11.12.42 # Hostname/IP of the first data node
NodeId=2                        # Node ID for this data node
datadir=/usr/local/mysql/data   # Remote directory for the data files

[ndbd]
hostname=10.11.12.43 # Hostname/IP of the second data node
NodeId=3                        # Node ID for this data node
datadir=/usr/local/mysql/data   # Remote directory for the data files

[mysqld]
# SQL node options:
hostname=10.11.12.41 # In our case the MySQL server/client is on the same Droplet as the cluster manager
```
Start the manager
```
sudo ndb_mgmd -f /var/lib/mysql-cluster/config.ini
```
Kill the running server to create service
```
sudo pkill -f ndb_mgmd
```

Edit systemd Unit
```
sudo vi /etc/systemd/system/ndb_mgmd.service
```
Paste in the following code:
```
[Unit]
Description=MySQL NDB Cluster Management Server
After=network.target auditd.service

[Service]
Type=forking
ExecStart=/usr/sbin/ndb_mgmd -f /var/lib/mysql-cluster/config.ini
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Execute to start `ndb_mgmd`
```
sudo systemctl daemon-reload
sudo systemctl enable ndb_mgmd
sudo systemctl start ndb_mgmd
sudo systemctl status ndb_mgmd
```
Add rules to allow local incoming connections from both data nodes
```
sudo ufw allow from 10.11.12.42
sudo ufw allow from 10.11.12.43
```
## Step 2 — Installing and Configuring the Data Nodes
_Execute step 2 into both data node `10.11.12.42` and `10.11.12.43`_

Install dependency `libclass-methodmaker-perl` and the data note binary using `dpkg`.
```
sudo apt update
sudo apt install libclass-methodmaker-perl -y
wget https://cdn.mysql.com//Downloads/MySQL-Cluster-8.0/mysql-cluster-community-data-node_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-cluster-community-data-node_8.0.31-1ubuntu22.04_amd64.deb
```
Create data nodes configuration file
```
sudo vi /etc/my.cnf
```
Paste in the following code:
```
[mysql_cluster]
# Options for NDB Cluster processes:
ndb-connectstring=10.11.12.41  # location of cluster manager
```
Create data directory on the node
```
sudo mkdir -p /usr/local/mysql/data
```
start the data node
```
sudo ndbd
```
Add rules to allow incoming connections
```
sudo ufw allow from 10.11.12.41
sudo ufw allow from 10.11.12.43
```
Kill the running `ndbd` process to create service
```
sudo pkill -f ndbd
```

Edit systemd Unit
```
sudo vi /etc/systemd/system/ndb_mgmd.service
```
Paste in the following code:
```
[Unit]
Description=MySQL NDB Data Node Daemon
After=network.target auditd.service

[Service]
Type=forking
ExecStart=/usr/sbin/ndbd
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Execute to start `ndbd`
```
sudo systemctl daemon-reload
sudo systemctl enable ndbd
sudo systemctl start ndbd
sudo systemctl status ndbd
```

## Step 3 — Configuring and Starting the MySQL Server and Client
_Execute step 3 into Cluster Manager server `10.11.12.41`. In a production environment running these daemons on different nodes is recommended._

Download and extract `.tar` archive into `install` directory
```
wget https://cdn.mysql.com//Downloads/MySQL-Cluster-8.0/mysql-cluster_8.0.31-1ubuntu22.04_amd64.deb-bundle.tar
mkdir install
tar -xvf mysql-cluster_8.0.31-1ubuntu22.04_amd64.deb-bundle.tar -C install/
```
Install dependencies `libaio1` `libmecab2`
```
sudo apt update
sudo apt install -y libaio1 libmecab2
```

Install the MySQL Cluster dependencies, bundled in the tar archive
```
cd install
sudo dpkg -i mysql-common_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-cluster-community-client-plugins_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-cluster-community-client-core_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-cluster-community-client_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-client_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-cluster-community-server-core_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-cluster-community-server_8.0.31-1ubuntu22.04_amd64.deb
sudo dpkg -i mysql-server_8.0.31-1ubuntu22.04_amd64.deb
```
