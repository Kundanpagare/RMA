# SLURM Installation
* Make 3 vm Machine (One is Master & 2 is Nodes)with 2 Network Adapter (NAT & HOST-ONLY). 
* *Now you need to configure **NFS** on Master & Nodes*

### ***On All 3 Machine***
-> Make Firewalld & SELINUX disabled by,
```bash
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld

setenforce 0
sed -i 's/^SELINUX=.*$/SELINUX=disabled/g'
getenforce
```

## Configure NFS 
  On Base Machine--> Search for **NFS On Centos 7**, then go to **itzgreek** website & from there follow the steps of NFS.

### ***On Master -***
-> Install the nfs package
```bash
yum install -y nfs-utils
```
-> Start the NFS Services
```
systemctl start nfs-server
systemctl enable nfs-server
systemctl status nfs-server
``` 

-> Make key less SSH on all three machines.
```bash
SSH-keygen
cd .ssh
copy-id root@<nodes_ip>
```
->Allow NFS client to read write permission to the created directory
```bash
chmod 777 /home/
```
-> Make entry of the standard file in the /etc/exports
```bash
vim /etc/exports
```
-> Create a NFS share something like below & add this entry to above file.
```bash
/home <nodes_ip_network>(rw,sync,no_root_squash) 
```
-> Export the shared directories using the following command.
```bash
exportfs -r
```

## ***On Clients -***
##### (Now Configure NFS on Both Clients)

-> Install the Packages on clients
```bash
yum install -y nfs-utils
```
-> Check if Shared file or directory is visible or not
```bash
showmount -e <master_host-only_ip>
```
-> Mount the Shared Directory
```bash
mount -t nfs <maste_host_ip>:/home /home
```
-> To check if the given directory is mounted or not
```bash
mount | grep nfs

df -hT
```
-> how to enable the automount
```bash
echo <master_host_ip>:/home /home nfs nosuid,rw,sync,hard,intr 0 0 >> vim /etc/fstab
```

-> How to unmount the shared directory (optional)
```bash
umount /home
```

***On All 3 Machines(Master & Nodes) -***

-> Make host entry on each machine as below
```bash
vim /etc/hosts
```
*make this below entry in above file*
```
<master_ip> <hostname>
<node1_ip> <node1_hostname>
<node2_ip> <node2_hostname>
```
-> Install the below package on each machine
```bash
yum install epel-release
```
-> Install the **Munge** packages on each machines
```bash
yum install munge munge-libs munge-devel
```
-> To check the packages 
```bash
rpm -qa | grep munge
```
### **On Master** -
-> To create the **Munge-Key** & then check it as
```bash
/usr/sbin/create-munge-key -r
ll /etc/munge
```
->  Now Copy the munge-key from master to clients & check it by 
```bash
scp /etc/munge/munge.key <node1_hostname>:/etc/munge
scp /etc/munge/munge.key <node2_hostname>:/etc/munge
ls /etc/munge
```
-> Start the Munge services by 
```bash
systemctl start munge.service
systemctl enable munge
systemctl status munge 
```
### **On Client** -
-> Change the permission of munge key by 
```bash
chown munge:munge /etc/munge/munge.key
```
-> Then start the munge service 
```bash
systemctl start munge.service
systemctl enable munge
systemctl status munge 
```
### **On Master** - 
-> Download the Slurm Package as 
```bash
wget https://download.schedmd.com/slurm/slurm-20.11.9.tar.bz2
```
-> install the below packages 
```bash
yum install pam-devel python3 readline-devel perl-ExtUtils-MakeMaker gcc -y
```
-> We need to install rmp-build to convert the tar file into rpm by 
```bash
yum install rpm-build
rpmbuild -ta slurm-20.11.9.tar.bz2
```
### **On Clients** -
-> install the below packages on both the nodes
```bash
yum install pam-devel python3 readline-devel perl-ExtUtils-MakeMaker gcc mysql-devel -y
```
### **On All 3 machines** -
  -> To export an environment variable
```bash
export SLURMUSER=900
```
-> To create the group user
```bash
groupadd -g $GROUPID <name>

for e.g. groupadd -g $GROUPID slurm
where, -g = groupid
```
-> To create a user in slurm 
```bash
useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm

where, -m = create user home directory
       -c = comments " "
       -d = specify the user home directory <diretory_path>
       -u = userID $
       -g = to specify the groupid <name>
       -s = shell login <shell_login>
```
-> To check the user is added or not in 
```bash
cat /etc/passwd
```
-> To check the rpmbuild
```bash
ls /root/rpmbuild/RPMS/x86_64
```
### **On Master** -
-> To shift the nfs, we make a directory
```bash
mkdir /home/rpms
cd /root/rpmbuild/RPMS/x86_64/
ls
```
-> Then copy the rpm to that directory
```bash
cp * /home/rpms/
cd /home/rpms
yum --nogpgcheck localinstall * -y
```

### **On Clients** -
-> We need to remove 2 packages, so
```bash
cd /home/rpms
ls
rm -rf slurm-slurmctld-20.11.9-1.e17.x86_64.rpm
rm -rf slurm-slurmdbd-20.11.9-1.e17.x86_64.rpm
yum --nogpgcheck localinstall * -y
```
### **On All 3 Machines** - 
-> Now check on each machine how many packages are installed. 
```bash
rpm -qa | grep slurm | wc -l
```
It should be shown as below
- Master = 12 Packages
- Nodes = 10 Packages 
  
-> Make the below directory
```bash
mkdir /var/spool/slurm
```
-> Change the permission of created directory from root to slurm & chek it by,
```bash
chown slurm:slurm /var/spool/slurm
ll /var/spool
``` 
-> Now change the permission of slurm by,
```bash
chmod 755 /var/spool/slurm
```
-> Make a directory of logs & check the permission
```bash
mkdir /var/log/slurm
chown -R slurm . /var/log/slurm
```
### **On Master** -
-> Make a file  & then  change the permission by,
```bash 
touch /var/log/slurm/slurmctld.log
chown slurm:slurm /var/log/slurm/slurmctld.log
```
-> Make 2 log file for Job Accounting & Job Completion & then change the permission
```bash
touch /var/log/slurm_jobacct.log /var/logslurm_jobcomp.log
chown slurm: . /var/log/slurm_jobacct.log /var/logslurm_jobcomp.log
```
-> There is one file name as slurm.conf.example & now we will change the filee name as 
```bash
cp /etc/slurm/slurm.conf.example /etc/slurm/slurm.conf
```
## **On Clients** -
-> Copy the output of below command & paste into the master slurm.conf file
```bash
slurmd -C
```
-> Start the slurmd services
```bash
systemctl start slurmd
systemctl enable slurmd
systemctl status slurmd
```
## **On Master** -
-> Now edit the slurm.conf file as below
  
  - In Compute nodes (at last) comment the ***#NodeName=linux[1-32]*** line & then add 2 lines below it.
  - paste the lines from Both Nodes & add **State=UNKNOWN** at last
```bash
vim /etc/slurm/slurm.conf
```
-> We need to configure this file on Nodes also, so
```bash
scp /etc/slurm/slurm.conf <Node1_hostname>:/etc/slurm/
scp /etc/slurm/slurm.conf <Node2_hostname>:/etc/slurm/
```
-> Now start the slurmctld service by
```bash
systemctl start slurmctld 
systemctl enable slurmctld
systemctl status slurmctld
```
-> To cheeck the information about the current state and status of the nodes 
```bash
sinfo
```
- If the state of the node is down the update the state by 
```bash
scontrol update node=<node_name> state=idle
```
-> For Debugging 
```bash
slurmctld -Dvv
```
-> To display the  extended information about the resources available on the nodes
```bash
sinfo -R
```
-> To run the job, 
```bash
srun -w <node_hostname> --pty /bin/bash

OR

srun -N1 --pty /bin/bash
``` 
-> If you want to down a Node for any reason, then you need to specify reason
```bash
scontrol updatenode=<node_hostname> state=down reason=<reason>

for e.g.- scontrol updatenode=<node_hostname> state=down reason=maintenance
```
-> Create one file with .sh & write the slurm script in that file
```bash
vim demo.sh
```
- write the below script in above file
  ```bash
  #!/bin/bash
  #SBATCH --job-name=myjob
  #SBATCH --partition=standard
  #SBATCH --nodes=2
  #SBATCH --ntasks-per-node=2
  #SBATCH --cpus-per-task=2
  #SBATCH --time=10-01:30:00    ......(for how many time it should be run)
  sleep 3000
  ```
-> To submit the Job
  ```bash
  sbatch demo.sh
  ```
  -> To check the queues
  ```bash
  squeue
  ```
  -> To see the Job Details
  ```bash
  scontrol show job <Job_id>
  ```
  -> To cancel the Job
  ```bash
  scancel <job_id>
  ```
  -> To see the Account
  ```bash
  sshare
  ```

