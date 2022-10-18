# Create a Multi-Node MySQL Cluster on Ubuntu 22.04

In this LAB will need a total of three servers: two servers for the redundant MySQL data nodes (`ndbd`), and one server for the Cluster Manager (`ndb_mgmd`) and MySQL server/client (`mysqld` and `mysql`).

- `10.11.12.41` will be the Cluster Manager & MySQL server node
- `10.11.12.42` will be the first MySQL Cluster data node
- `10.11.12.43` will be the second data node

[Official MySQL Cluster download page](https://dev.mysql.com/downloads/cluster/)
