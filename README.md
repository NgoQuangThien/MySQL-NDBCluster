#   Setup MySQL NDB Cluster
##  All Node
```
tee -a /etc/hosts <<EOF
192.168.100.20          mgm_1
192.168.100.21          mgm_2

192.168.100.22          ndb_1
192.168.100.23          ndb_2

192.168.100.20          api_1
192.168.100.21          api_2
EOF
```
```
git clone https://github.com/NgoQuangThien/MySQL-NDBCluster.git
```
##	Management Node
```
sudo dpkg -i mysql-cluster-community-management-server_8.0.30-1ubuntu20.04_amd64.deb
```
```
mkdir -p /var/lib/mysql-cluster
nano /var/lib/mysql-cluster/config.ini
```
```
[tcp default]
SendBufferMemory=3M
ReceiveBufferMemory=3M
# Increasing the sizes of these 2 buffers beyond the default values
# helps prevent bottlenecks due to slow disk I/O.

[ndb_mgmd default]
DataDir=/var/lib/mysql-cluster  # Directory for management node log files

[ndb_mgmd]
# Management process options:
HostName=mgm_1             # Hostname or IP address of management node
NodeId=1                        # Node ID for this manage node

[ndb_mgmd]
# Management process options:
HostName=mgm_2             # Hostname or IP address of management node
NodeId=2                        # Node ID for this manage node

[ndbd default]
# Options affecting ndbd processes on all data nodes:
NoOfReplicas=2    # Number of fragment replicas
DataMemory=98M    # How much memory to allocate for data storage

DataDir=/usr/local/mysql/data   # Directory for this data node`s data files

LockPagesInMainMemory=1
# On Linux and Solaris systems, setting this parameter locks data node
# processes into memory. Doing so prevents them from swapping to disk,
# which can severely degrade cluster performance.

#DataMemory=3456M

# The value provided for DataMemory assumes 4 GB RAM
# per data node. However, for best results, you should first calculate
# the memory that would be used based on the data you actually plan to
# store (you may find the ndb_size.pl utility helpful in estimating
# this), then allow an extra 20% over the calculated values. Naturally,
# you should ensure that each data node host has at least as much
# physical memory as the sum of these two values.

ODirect=1
# Enabling this parameter causes NDBCLUSTER to try using O_DIRECT
# writes for local checkpoints and redo logs; this can reduce load on
# CPUs. We recommend doing so when using NDB Cluster on systems running
# Linux kernel 2.6 or later.

#SchedulerSpinTimer=400
#SchedulerExecutionTimer=100
#RealTimeScheduler=1
# Setting these parameters allows you to take advantage of real-time scheduling
# of NDB threads to achieve increased throughput when using ndbd. They
# are not needed when using ndbmtd; in particular, you should not set
# RealTimeScheduler for ndbmtd data nodes.

#TimeBetweenGlobalCheckpoints=1000
#TimeBetweenEpochs=200
#RedoBuffer=32M

# (one [ndbd] section per data node)
[ndbd]
# Options for data node "A":
HostName=ndb_1                 # Hostname or IP address
NodeId=11                       # Node ID for this data node

#LockExecuteThreadToCPU=1
#LockMaintThreadsToCPU=0
# On systems with multiple CPUs, these parameters can be used to lock NDBCLUSTER
# threads to specific CPUs

[ndbd]
# Options for data node "B":
HostName=ndb_2                 # Hostname or IP address
NodeId=12                       # Node ID for this data node

#LockExecuteThreadToCPU=1
#LockMaintThreadsToCPU=0

[mysqld]
# SQL node options:
HostName=api_1               # Hostname or IP address
NodeId=21                       # Node ID for this sql node
                                # (additional mysqld connections can be
                                # specified for this node for various
                                # purposes such as running ndb_restore)

[mysqld]
# SQL node options:
HostName=api_2               # Hostname or IP address
NodeId=22                       # Node ID for this sql node
                                # (additional mysqld connections can be
                                # specified for this node for various
                                # purposes such as running ndb_restore)
```
```
ndb_mgmd --initial -f /var/lib/mysql-cluster/config.ini
```
```
sudo pkill -f ndb_mgmd
```
```
sudo nano /etc/systemd/system/ndb_mgmd.service
```
```
[Unit]
Description=MySQL NDB Cluster Management Server
After=network.target auditd.service

[Service]
Type=forking
ExecStart=/usr/sbin/ndb_mgmd --reload -f /var/lib/mysql-cluster/config.ini
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl daemon-reload
```
```
sudo systemctl enable ndb_mgmd
```
```
sudo systemctl start ndb_mgmd
```
```
sudo systemctl status ndb_mgmd
```

##  Data Node
```
sudo apt-get install libclass-methodmaker-perl
```
```
sudo dpkg -i mysql-cluster-community-data-node_8.0.30-1ubuntu20.04_amd64.deb
```
```
sudo nano /etc/my.cnf
```
```
[mysql_cluster]
# Options for NDB Cluster processes:
ndb-connectstring=mgm_1,mgm_2  # location of management server
```
```
sudo mkdir -p /usr/local/mysql/data
```
```
sudo ndbd
```
```
sudo pkill -f ndbd
```
```
sudo nano /etc/systemd/system/ndbd.service
```
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
```
sudo systemctl daemon-reload
```
```
sudo systemctl enable ndbd
```
```
sudo systemctl start ndbd
```
```
sudo systemctl status ndbd
```

##  SQL Node
```
sudo apt-get install libaio1 libmecab2
```
```
sudo dpkg -i mysql-common_8.0.30-1ubuntu20.04_amd64.deb
```
```
sudo dpkg -i mysql-cluster-community-client-plugins_8.0.30-1ubuntu20.04_amd64.deb mysql-cluster-community-client-core_8.0.30-1ubuntu20.04_amd64.deb mysql-cluster-community-client_8.0.30-1ubuntu20.04_amd64.deb mysql-client_8.0.30-1ubuntu20.04_amd64.deb
```
```
sudo dpkg -i mysql-cluster-community-server-core_8.0.30-1ubuntu20.04_amd64.deb mysql-cluster-community-server_8.0.30-1ubuntu20.04_amd64.deb mysql-server_8.0.30-1ubuntu20.04_amd64.deb
```
```
sudo nano /etc/mysql/my.cnf
```
```
[mysqld]
# Options for mysqld process:
ndbcluster                      # run NDB storage engine

[mysql_cluster]
# Options for NDB Cluster processes:
ndb-connectstring=mgm_1,mgm_2  # location of management server
```
```
sudo systemctl restart mysql
```
```
sudo systemctl status mysql
```
```
sudo systemctl enable mysql
```

##  Verifying MySQL Cluster Installation
```
mysql -u root -p
SHOW ENGINE NDB STATUS;
```
```
ndb_mgm -e show
```
```
1 STATUS
```
