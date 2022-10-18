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
