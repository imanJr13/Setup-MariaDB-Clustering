# Setup-MariaDB-Clustering

## Introduction
Clustering adds high availability to your database by distributing changes to different servers. In the event that one of the instances fails, others are quickly available to continue serving.

Clusters come in two general configurations, active-passive and active-active. In active-passive clusters, all writes are done on a single active server and then copied to one or more passive servers that are poised to take over only in the event of an active server failure. Some active-passive clusters also allow SELECT operations on passive nodes. In an active-active cluster, every node is read-write and a change made to one is replicated to all.

MariaDB is an open source relational database system that is fully compatible with the popular MySQL RDBMS system. You can read the official documentation for MariaDB at this page. Galera is a database clustering solution that enables you to set up multi-master clusters using synchronous replication. Galera automatically handles keeping the data on different nodes in sync while allowing you to send read and write queries to any of the nodes in the cluster. You can learn more about Galera at the official documentation page.

In this guide, you will configure an active-active MariaDB Galera cluster. For demonstration purposes, you will configure and test three CentOS 7 Droplets that will act as nodes in the cluster. This is the smallest configurable cluster.

# Prerequisites
To follow along, you will need a DigitalOcean account, in addition to the following:

Three CentOS 7 Droplets with private networking enabled, each with a non-root user with sudo privileges and a firewall enabled.
For setting up private networking on the three Droplets, follow our Private Networking Quickstart guide.
For assistance setting up a non-root user with sudo privileges, follow our Initial Server Setup with CentOS 7 tutorial. To set up a firewall, check out the Configuring a Basic Firewall step of Additional Recommended Steps for New CentOS 7 Servers.
While the steps in this tutorial have been written for and tested against DigitalOcean Droplets, many of them should also be applicable to non-DigitalOcean servers with private networking enabled.

# Step 1 — Adding the MariaDB Repositories to All Servers
In this step, you will add the relevant MariaDB package repositories to each of your three servers so that you will be able to install the right version of MariaDB used in this tutorial. Once the repositories are updated on all three servers, you will be ready to install MariaDB.

One thing to note about MariaDB is that it originated as a drop-in replacement for MySQL, so in many configuration files and startup scripts, you’ll see mysql rather than mariadb. In many cases, these are interchangeable. For consistency’s sake, we will use mariadb in this guide where either could work.

In this tutorial, you will use MariaDB version 10.4. Since this version isn’t included in the default CentOS repositories, you’ll start by adding the external CentOS repository maintained by the MariaDB project to all three of your servers.

```
sudo vi /etc/yum.repos.d/mariadb.repo
```

Next, add the following contents to the file by pressing i to enter insert mode, then adding the following:

```
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.4/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

Press the esc key to return to normal mode, then type :wq to save and exit the file. If you would like to learn more about the text editor vi and its predecessor vim, take a look at our tutorial on Installing and Using the Vim Text Editor on a Cloud Server.

Once you have created the repository file, enable it with the following command:

```
sudo yum makecache --disablerepo='*' --enablerepo='mariadb'
```
The makecache command caches the repository metadata so that the package manager can install MariaDB, with --disablerepo and --enablerepo targeting the command to the mariadb repo file that you just created.

You will receive the following output:

```
Output
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
mariadb                                                                                                                                                                              | 2.9 kB  00:00:00
(1/3): mariadb/primary_db                                                                                                                                                            |  43 kB  00:00:00
(2/3): mariadb/other_db                                                                                                                                                              | 8.3 kB  00:00:00
(3/3): mariadb/filelists_db                                                                                                                                                          | 238 kB  00:00:00
Metadata Cache Created
```
Once you have enabled the repository on your first server, repeat for your second and third servers.

Now that you have successfully added the package repository on all three of your servers, you’re ready to install MariaDB in the next section.
Step 2 — Installing MariaDB on All Servers
In this step, you will install the actual MariaDB packages on your three servers.

Beginning with version 10.1, the MariaDB Server and MariaDB Galera Server packages are combined, so installing MariaDB-server will automatically install Galera and several dependencies:

```
sudo yum install MariaDB-server MariaDB-client
```
You will be asked to confirm whether you would like to proceed with the installation. Enter yes to continue with the installation. You will then be prompted to accept the GPG key that authenticates the MariaDB package. Enter yes again.

When the installation is complete, start the mariadb service by running:

```
sudo systemctl start mariadb
```

Enable the mariadb service to be automatically started on boot by executing:

```
sudo systemctl enable mariadb
```
From MariaDB version 10.4 onwards, the root MariaDB user does not have a password by default. To set a password for the root user, start by logging into MariaDB:

```
sudo mysql -uroot
```
Once you’re inside the MariaDB shell, change the password by executing the following statement, replacing your_password with your desired password:

```
set password = password("your_password");
```

You will see the following output indicating that the password was set correctly:

```
Output
Query OK, 0 rows affected (0.001 sec)
```
Exit the MariaDB shell by running the following command:

```
quit;
```
If you would like to learn more about SQL or need a quick refresher, check out our MySQL tutorial.

You now have all of the pieces necessary to begin configuring the cluster, but since you’ll be relying on rsync and policycoreutils-python in later steps to sync the servers and to control Security-Enhanced Linux (SELinux), make sure they’re installed before moving on:

sudo yum install rsync policycoreutils-python

```
sudo yum install rsync policycoreutils-python
```

This will confirm that the newest versions of rsync and policycoreutils-python is already available or will prompt you to upgrade or install it.

Once you have completed these steps, repeat them for your other two servers.

Now that you have installed MariaDB successfully on each of the three servers, you can proceed to the configuration step in the next section.

# Step 3 — Configuring the First Node
In this step you will configure your first Galera node. Each node in the cluster needs to have a nearly identical configuration. Because of this, you will do all of the configuration on your first machine, and then copy it to the other nodes.

By default, MariaDB is configured to check the /etc/mysql/conf.d directory to get additional configuration settings from files ending in .cnf. Create a file in this directory with all of your cluster-specific directives:

```
sudo vi /etc/my.cnf.d/galera.cnf
```
Add the following configuration into the file. The configuration specifies different cluster options, details about the current server and the other servers in the cluster, and replication-related settings. Note that the IP addresses in the configuration are the private addresses of your respective servers; replace the highlighted lines with the appropriate IP addresses:

```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="test_cluster"
wsrep_cluster_address="gcomm://First_Node_IP,Second_Node_IP,Third_Node_IP"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="This_Node_IP"
wsrep_node_name="This_Node_Name"
```
The first section modifies or re-asserts MariaDB/MySQL settings that will allow the cluster to function correctly. For example, Galera won’t work with MyISAM or similar non-transactional storage engines, and mysqld must not be bound to the IP address for localhost.
The “Galera Provider Configuration” section configures the MariaDB components that provide a WriteSet replication API. This means Galera in your case, since Galera is a wsrep (WriteSet Replication) provider. You specify the general parameters to configure the initial replication environment. This doesn’t require any customization, but you can learn more about Galera configuration options here.
The “Galera Cluster Configuration” section defines the cluster, identifying the cluster members by IP address or resolvable domain name and creating a name for the cluster to ensure that members join the correct group. You can change the wsrep_cluster_name to something more meaningful than test_cluster or leave it as-is, but you must update wsrep_cluster_address with the private IP addresses of your three servers.
The “Galera Synchronization Configuration” section defines how the cluster will communicate and synchronize data between members. This is used only for the state transfer that happens when a node comes online. For your initial setup, you are using rsync, because it’s commonly available and does what you’ll need for now.
The “Galera Node Configuration” section clarifies the IP address and the name of the current server. This is helpful when trying to diagnose problems in logs and for referencing each server in multiple ways. The wsrep_node_address must match the address of the machine you’re on, but you can choose any name you want in order to help you identify the node in log files.

When you are satisfied with your cluster configuration file, copy the contents into your clipboard and save and close the file.

Now that you have configured your first node successfully, you can move on to configuring the remaining nodes in the next section.
# Step 4 — Configuring the Remaining Nodes
In this step, you will configure the remaining two nodes. On your second node, open the configuration file:
```
sudo vi /etc/mysql/my.cnf.d/galera.cnf
```
Paste in the configuration you copied from the first node, then update the Galera Node Configuration to use the IP address or resolvable domain name for the specific node you’re setting up. Finally, update its name, which you can set to whatever helps you identify the node in your log files:

```
. . .
# Galera Node Configuration
wsrep_node_address="This_Node_IP"
wsrep_node_name="This_Node_Name"
. . .
```
Save and exit the file.

Once you have completed these steps, repeat them on the third node.

With Galera configured on all of your nodes, you’re almost ready to bring up the cluster. But before you do, make sure that the appropriate ports are open in your firewall and that a SELinux policy has been created for Galera.

Step 5 — Opening the Firewall on Every Server
In this step, you will configure your firewall so that the ports required for inter-node communication are open.

On every server, check the status of the firewall you set up in the Prerequisites section by running:

```
sudo firewall-cmd --list-all
```
In this case, only SSH, DHCP, HTTP, and HTTPS traffic is allowed through:
```
Output
public
  target: default
  icmp-block-inversion: no
  interfaces:
  sources:
  services: ssh dhcpv6-client http https
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
If you tried to start the cluster now, it would fail because the firewall would block the connections between the nodes. To solve this problem, add rules to allow MariaDB and Galera traffic through.

Galera can make use of four ports:

3306 For MariaDB client connections and State Snapshot Transfer that use the mysqldump method.
4567 For Galera Cluster replication traffic. Multicast replication uses both UDP transport and TCP on this port.
4568 For Incremental State Transfers, or IST, the process by which a missing state is received by other nodes in the cluster.
4444 For all other State Snapshot Transfers, or SST, the mechanism by which a joiner node gets its state and data from a donor node.
In this example, you’ll open all four ports while you do your setup. Once you’ve confirmed that replication is working, you’d want to close any ports you’re not actually using and restrict traffic to just servers in the cluster.

Open the ports with the following commands:

```
sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp
sudo firewall-cmd --permanent --zone=public --add-port=4567/tcp
sudo firewall-cmd --permanent --zone=public --add-port=4568/tcp
sudo firewall-cmd --permanent --zone=public --add-port=4444/tcp
sudo firewall-cmd --permanent --zone=public --add-port=4567/udp
```
Using --zone=public and --add-port= here, firewall-cmd is opening up these ports to public traffic. --permanent ensures that these rules persist.
