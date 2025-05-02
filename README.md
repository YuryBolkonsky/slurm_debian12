# Slurm Debian12 Installation Guide

## 1️⃣ Requirements (before installation)

- N nodes with **Debian 12** (example: `g1`, `g2`, `g3`, `g4`, ...)
- **Synchronized UID/GIDs** across all nodes
    - Manual sync (if no frequent user changes)
    - Or use **FreeIPA** (if frequent create/delete/modify of users)
- **Shared folder** between all nodes (e.g., `/m2` via NFS/Lustre)
- **NVIDIA GPU Drivers** and any other required drivers installed on each node
- **Passwordless SSH** access from **master node** to all other nodes

### Create users and groups (run on **all nodes**)

```bash
sudo addgroup -gid 1111 munge
sudo addgroup -gid 1121 slurm
sudo adduser -u 1111 munge --disabled-password --gecos "" -gid 1111
sudo adduser -u 1121 slurm --disabled-password --gecos "" -gid 1121
```

---

## 2️⃣ Setup munge

### On **master node**:

```bash
sudo apt-get install libmunge-dev libmunge2 munge
sudo /usr/sbin/create-munge-key
sudo chown munge: /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
scp /etc/munge/munge.key root@g1:/etc/munge
scp /etc/munge/munge.key root@g2:/etc/munge
```

### On **worker nodes (g1, g2, ...)**:

```bash
sudo chown -R munge: /etc/munge/ /var/log/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/
sudo systemctl enable munge
sudo systemctl start munge
sudo systemctl status munge
```

Test:

```bash
munge -n
munge -n | munge
munge -n | ssh g2 unmunge
munge -n | ssh g3 unmunge
```

---

## 3️⃣ Install Slurm (run on **all nodes**)

```bash
sudo apt-get install git gcc make ruby ruby-dev libpam0g-dev mariadb-server build-essential libssl-dev libmariadb-dev libdbus-1-dev linux-headers-amd64 -y
sudo gem install fpm

wget https://download.schedmd.com/slurm/slurm-xx.xx.x.tar.bz2
tar xvjf slurm-xx.xx.x.tar.bz2
cd slurm-xx.xx.x
./configure --prefix=/usr/local/slurm --sysconfdir=/etc/slurm --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm
make
make contrib
make install
cd ..

sudo fpm -s dir -t deb -v 1.0 -n slurm-xx.xx.x --prefix=/usr -C /usr/local/slurm .
sudo dpkg -i slurm-xx.xx.xx_1.0_amd64.deb
scp /usr/local/slurm g1:/usr/local/slurm
```

Setup directories:

```bash
sudo mkdir -p /etc/slurm /etc/slurm/prolog.d /etc/slurm/epilog.d /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm
sudo chown slurm /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm
```

Create tmpfiles config:

```bash
sudo nano /etc/tmpfiles.d/slurm.conf
# content:
d /run/slurm 0770 root slurm -
```

---

## 4️⃣ Configure Database (**master node only**)

Create `/etc/slurm/slurmdbd.conf`:

```ini
PurgeEventAfter=1month
PurgeJobAfter=12month
PurgeResvAfter=1month
PurgeStepAfter=1month
PurgeSuspendAfter=1month
PurgeTXNAfter=12month
PurgeUsageAfter=24month

AuthType=auth/munge

DbdAddr=localhost
DbdHost=localhost
SlurmUser=slurm
DebugLevel=verbose
DefaultQOS=normal
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurm/slurmdbd.pid
PluginDir=/usr/lib64/slurm
PrivateData=accounts,users,usage,jobs

StorageType=accounting_storage/mysql
StoragePass=your_password
StorageUser=slurm
```

```bash
sudo chown slurm: /etc/slurm/slurmdbd.conf
sudo chmod 600 /etc/slurm/slurmdbd.conf

sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb
```

Setup MariaDB:

```sql
mysql
> GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost' IDENTIFIED BY '1234' WITH GRANT OPTION;
> SHOW VARIABLES LIKE 'have_innodb';
> FLUSH PRIVILEGES;
> CREATE DATABASE slurm_acct_db;
> quit;
```

---

## 5️⃣ Configure systemd services

### `/etc/systemd/system/slurmdbd.service` (master node):

```ini
[Unit]
Description=Slurm DBD accounting daemon
After=network.target munge.service
ConditionPathExists=/etc/slurm/slurmdbd.conf

[Service]
Type=forking
EnvironmentFile=-/etc/sysconfig/slurmdbd
ExecStart=/usr/sbin/slurmdbd $SLURMDBD_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/run/slurm/slurmdbd.pid

[Install]
WantedBy=multi-user.target
```

### `/etc/systemd/system/slurmd.service` (**all nodes**):

```ini
[Unit]
Description=Slurm node daemon
After=network.target munge.service slurmctld.service
ConditionPathExists=/etc/slurm/slurm.conf

[Service]
Type=forking
EnvironmentFile=-/etc/sysconfig/slurmd
ExecStart=/usr/sbin/slurmd -d /usr/sbin/slurmstepd $SLURMD_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/run/slurm/slurmd.pid
KillMode=process
LimitNOFILE=51200
LimitMEMLOCK=infinity
LimitSTACK=infinity

[Install]
WantedBy=multi-user.target
```

### `/etc/systemd/system/slurmctld.service` (**master node**):

```ini
[Unit]
Description=Slurm controller daemon
After=network.target munge.service
ConditionPathExists=/etc/slurm/slurm.conf

[Service]
Type=forking
EnvironmentFile=-/etc/sysconfig/slurmctld
ExecStart=/usr/sbin/slurmctld $SLURMCTLD_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/run/slurm/slurmctld.pid

[Install]
WantedBy=multi-user.target
```

---

## 6️⃣ Configure Slurm configs

Create `/etc/slurm/cgroup_allowed_devices_file.conf`:

```bash
/dev/null
/dev/urandom
/dev/zero
/dev/sda*
/dev/cpu/*/*
/dev/pts/*
/dev/nvidia*
```

Create `/etc/slurm/cgroup.conf`:

```ini
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
```

Create `/etc/slurm/slurm.conf` (use your settings):

```ini
ClusterName=hpc_cluster
SlurmctldHost=g3
SlurmUser=slurm
#SlurmctldPort=6817
#SlurmdPort=6818
AuthType=auth/munge
#JobCredentialPrivateKey=
#JobCredentialPublicCertificate=
StateSaveLocation=/var/spool/slurm/ctld
SlurmdSpoolDir=/var/spool/slurm/d
SwitchType=switch/none
MpiDefault=pmi2
SlurmctldPidFile=/run/slurm/slurmctld.pid
SlurmdPidFile=/run/slurm/slurmd.pid
ProctrackType=proctrack/cgroup
PluginDir=/usr/lib/slurm
#FirstJobId=
ReturnToService=2
#MaxJobCount=10000
#PlugStackConfig=
#PropagatePrioProcess=
#PropagateResourceLimits=
PropagateResourceLimitsExcept=MEMLOCK
#Prolog=/etc/slurm/prolog.d/*
#Epilog=/etc/slurm/epilog.d/*
#SrunProlog=
#SrunEpilog=
#TaskProlog=
#TaskEpilog=
TaskPlugin=task/affinity,task/cgroup
TaskPluginParam=Cores
PrologFlags=contain
#TrackWCKey=no
#TreeWidth=50
#TmpFS=
UsePAM=1
#TopologyPlugin=topology/tree
RebootProgram=/usr/sbin/reboot
#
CpuFreqDef=Performance
DisableRootJobs=yes
EnforcePartLimits=all
#
# TIMERS
#SlurmctldTimeout=120
SlurmdTimeout=30
InactiveLimit=10
#MinJobAge=300
#KillWait=30
WaitTime=60
#
# SCHEDULING
SchedulerType=sched/backfill
PreemptType=preempt/qos
PreemptMode=REQUEUE
SelectType=select/cons_tres
SelectTypeParameters=CR_ONE_TASK_PER_CORE,CR_Core_Memory
PriorityType=priority/multifactor
PriorityDecayHalfLife=7-0
PriorityUsageResetPeriod=MONTHLY
PriorityWeightFairshare=10000
PriorityWeightAssoc=1000
PriorityWeightQOS=10
PriorityWeightPartition=10
PriorityWeightTRES=cpu=2000,mem=1000
PriorityWeightAge=1000
PriorityWeightJobSize=1000
#PriorityFavorSmall=YES
#PriorityMaxAge=1-0
#
# LOGGING
SlurmctldParameters=enable_configless
SlurmctldDebug=verbose
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdDebug=verbose
SlurmdLogFile=/var/log/slurm/slurmd.log
JobCompType=jobcomp/none
JobCompLoc=/var/log/slurm/jobcomp.log
#
# ACCOUNTING
#JobAcctGatherType=jobacct_gather/linux
#JobAcctGatherFrequency=30
#
#AccountingStorageType=accounting_storage/slurmdbd
#AccountingStorageEnforce=associations,limits,qos
#AccountingStorageHost=localhost
#AccountingStorageLoc=
#AccountingStoragePass=
#AccountingStorageUser=
#PrivateData=jobs,usage
#
# COMPUTE NODES
GresTypes=gpu
NodeName=g1 NodeAddr=192.168.2.12 Gres=gpu:2 CPUs=256 Boards=1 SocketsPerBoard=1 CoresPerSocket=128 ThreadsPerCore=2 RealMemory=1160365 Feature=2.25GHz State=UNKNOWN
NodeName=g2 NodeAddr=192.168.2.22 Gres=gpu:2 CPUs=256 Boards=1 SocketsPerBoard=1 CoresPerSocket=128 ThreadsPerCore=2 RealMemory=1160365 Feature=2.25GHz State=UNKNOWN
NodeName=g3 NodeAddr=192.168.2.32 CPUs=128 Boards=1 SocketsPerBoard=2 CoresPerSocket=32 ThreadsPerCore=2 RealMemory=515800 Feature=2.5GHz State=UNKNOWN
PartitionName=main Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

Copy configs to worker nodes:

```bash
scp -r /etc/slurm/cgroup_allowed_devices_file.conf /etc/slurm/cgroup.conf /etc/slurm/slurm.conf /etc/slurm/gres.conf g1:/etc/slurm/
```

---

## 7️⃣ Start services

### On **master node**:

```bash
sudo systemctl enable slurmdbd.service
sudo systemctl start slurmdbd.service
sudo systemctl status slurmdbd.service

sudo systemctl enable slurmctld.service
sudo systemctl start slurmctld.service
sudo systemctl status slurmctld.service
```

### On **all worker nodes (and master if it’s also compute)**:

```bash
sudo systemctl enable slurmd.service
sudo systemctl start slurmd.service
sudo systemctl status slurmd.service
```

---

## 8️⃣ Configure logrotate

Create `/etc/logrotate.d/slurm`:

```bash
/var/log/slurm/*.log {
    missingok
    notifempty
    copytruncate
}
```

Test:

```bash
logrotate -d /etc/logrotate.d/slurm
```

---

## 9️⃣ Testing

```bash
srun -N3 -l /bin/hostname
srun -N3 -l du -sh
srun --gres=gpu:2 -l nvidia-smi
srun --gres=gpu:1 -l nvidia-smi
```

---

Done. Thanks for you attention
