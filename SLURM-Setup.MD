# Enable communication between nodes
## Install SSH Server on all login nodes and compute nodes
```
sudo apt-get update
sudo apt-get install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
sudo ufw allow ssh
```

## Enable passwordless entry on compute nodes from login node
```
# On the login node, generate a new SSH key pair
ssh-keygen -t rsa

# Go to the .ssh directory
cd ~/.ssh

# Copy the public key to the compute node
ssh-copy-id user_id@hostname

# Test the connection
ssh user_id@hostname
```
## Set alias for the compute nodes
```
# Editing the hosts file
sudo nano /etc/hosts
ip_address_1 login_node
ip_address_2 compute_node_1
ip_address_3 compute_node_2
```

# Setup Network File System
## On the login node
```
# install nfs server
sudo apt-get install nfs-server

# create a directory to be shared
sudo mkdir /nfs

# edit the exports file
sudo nano /etc/exports

# add the following line at the end of the file
/nfs *(rw,sync)

# restart the nfs server
sudo service nfs-kernel-server restart

# change the owner of the directory to the user
ls -ld /nfs
sudo chown user_id /nfs

# adjust the firewall to open port 2049 on the host
sudo ufw allow from any to any port nfs
```

## On the compute nodes
```
# install nfs client
sudo apt-get install nfs-client

# create a directory to mount the shared directory
sudo mkdir /nfs

# edit file to mount the shared directory
sudo nano /etc/fstab

# add the following line at the end of the file
login_node:/nfs /nfs nfs

# reload the deamon
sudo systemctl daemon-reload

# mount the shared directory
sudo mount -a
```

## Test the NFS
```
# create a file in the shared directory on the login node
touch /nfs/test.txt

# check if the file is available on the compute node
ls /nfs
``` 

# Setup Slurm on the login node
## Create Munge User
```
export MUNGEUSER=1001
sudo groupadd -g $MUNGEUSER munge
sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge 
```
## Create Slurm User
```
export SLURMUSER=1002
sudo groupadd -g $SLURMUSER slurm
sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
```

## Install Munge
```
sudo apt-get install -y munge
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo scp /etc/munge/munge.key /nfs/slurm/
sudo chmod -R 777 /nfs/slurm/munge.key
sudo systemctl enable munge
sudo systemctl start munge
```

## Install slurm and associated components on slurm controller (Login) node
```
sudo apt-get install mariadb-server
sudo apt-get install slurmdbd
sudo apt-get install slurm-wlm
```

## Create and configure the slurm_acct_db database: (Login Node)
```
# Login as root
su

# Start mysql
mysql
create user 'slurm'@'localhost' identified by 'enter_password';
grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'enter_password' with grant option;
create database slurm_acct_db;
exit

# Create the slurmdbd.conf file by copying the following content
sudo nano /etc/slurm/slurmdbd.conf

AuthType=auth/munge
DbdAddr=localhost
DbdHost=localhost
DbdPort=6819
SlurmUser=slurm
DebugLevel=4
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurm/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageLoc=slurm_acct_db
StoragePass=enter_password
StorageUser=slurm
PurgeEventAfter=12months
PurgeJobAfter=12months
PurgeResvAfter=2months
PurgeStepAfter=2months
PurgeSuspendAfter=1month
PurgeTXNAfter=12months
PurgeUsageAfter=12months

# Change the owner of the file
chown slurm:slurm /etc/slurm/slurmdbd.conf
chmod -R 600 /etc/slurm/slurmdbd.conf
```

#### Configuration file /etc/slurm/slurm.conf:
```
# Visit the website (https://slurm.schedmd.com/configurator.html) to generate a slurm configuration file
Visit the website (https://slurm.schedmd.com/configurato...) to generate a slurm configuration file
sudo nano /etc/slurm/slurm.conf

# Allow the ports in firewall
sudo ufw allow 6817
sudo ufw allow 6818
sudo ufw allow 6819
```

## Make necessary folders and files
```
sudo mkdir /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmctld
sudo chmod 755 /var/spool/slurmctld

sudo mkdir /var/log/slurm
sudo touch /var/log/slurm/slurmctld.log
sudo touch /var/log/slurm/slurm_jobacct.log 
sudo touch /var/log/slurm/slurm_jobcomp.log
sudo chown -R slurm:slurm /var/log/slurm/
sudo chmod 755 /var/log/slurm


## Edit the slurmctld.service file and change PIDFile path to /run/slurm/slurmctld.pid
nano /usr/lib/systemd/system/slurmctld.service

## Edit the slurmdbd.service file and change PIDFile path to /run/slurm/slurmdbd.pid
nano /usr/lib/systemd/system/slurmdbd.service

## Edit the slurmd.service file and change PIDFile path to /run/slurm/slurmd.pid
nano /usr/lib/systemd/system/slurmd.service

## Run the following as root
echo CgroupMountpoint=/sys/fs/cgroup >> /etc/slurm/cgroup.conf

# Check the available compute nodes
slurmd -C
```

#### Start SLURM Services on Login Node
``` 
sudo systemctl daemon-reload
sudo systemctl enable slurmdbd
sudo systemctl start slurmdbd
sudo systemctl enable slurmctld
sudo systemctl start slurmctld
```
# Check the status of the services
```
systemctl status slurmdbd
systemctl status slurmctld
```

If any of the services are not active, try rebooting the PC and then check again. Hopefully that will do the job.

# Slurm setup on the Compute Nodes

## Create Munge User
```
export MUNGEUSER=1001 
sudo groupadd -g $MUNGEUSER munge 
sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge 
```

## Create Slurm User
```
export SLURMUSER=1002 
sudo groupadd -g $SLURMUSER slurm 
sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
```

## Install Munge
```
sudo apt-get install -y munge
```

## Copy the munge authentication key from /nfs/slurm/ on compute node.
```
sudo cp /nfs/slurm/munge.key /etc/munge/
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```
## Start Munge Service
```
sudo systemctl enable munge
sudo systemctl start munge
```

## Install Slurm
```
sudo apt-get install slurm-wlm
```

## Copy slurm.conf file from /nfs/slurm/ to /etc/slurm/
```
sudo cp /nfs/slurm/slurm.conf /etc/slurm/
```

## Create the required directory structure
```
sudo mkdir /var/spool/slurmd 
sudo chown slurm: /var/spool/slurmd
sudo chmod 755 /var/spool/slurmd

sudo mkdir /var/log/slurm/
sudo touch /var/log/slurm/slurmd.log
sudo chown -R slurm:slurm /var/log/slurm/slurmd.log
sudo chmod 755 /var/log/slurm

sudo mkdir /run/slurm
sudo touch /run/slurm/slurmd.pid
sudo chown slurm /run/slurm
sudo chown slurm:slurm /run/slurm
sudo chmod -R 770 /run/slurm
```

## Edit the slurmd.service file and change PIDFile path to /run/slurm/slurmd.pid
```
nano /usr/lib/systemd/system/slurmd.service

echo CgroupMountpoint=/sys/fs/cgroup >> /etc/slurm/cgroup.conf
slurmd -C
```

## Start the slurmd service
```
systemctl enable slurmd.service 
systemctl start slurmd.service 
systemctl status slurmd.service
```
If the service is active, you are all good, otherwise just reboot the node and reconnect the NFS. Then check the status of slurmd.service. In any case, a reboot at this stage is necessary.

## Check the connectivity with the controller node:
```
scontrol ping
```

# Common Commands for SLURM 

## Update the conf files
```
sudo nano /etc/slurm/slurm.conf
sudo nano /etc/slurm/slurmdbd.conf
```

## Copy slurm.conf file from /etc/slurm/ to /nfs/slurm/
```
sudo cp /etc/slurm/slurm.conf /nfs/slurm/
```

## Copy slurm.conf file from /nfs/slurm/ to /etc/slurm/
```
sudo cp /nfs/slurm/slurm.conf /etc/slurm/
```

## Restart the services
```
sudo systemctl restart slurmdbd
sudo systemctl restart slurmctld
sudo systemctl restart slurmd
```

## Check service status
```
systemctl status slurmdbd
systemctl status slurmctld
systemctl status slurmd
```

## View logs 
```
sudo cat /var/log/slurm/slurmdbd.log
sudo cat /var/log/slurm/slurmctld.log
sudo cat /var/log/slurm/slurmd.log
```
