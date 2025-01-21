# Slurm-101

Setting up a Slurm cluster can be challenging due to the limited detail in the [official instructions](https://slurm.schedmd.com/quickstart_admin.html#quick_start). This guide simplifies the process, focusing on configuring a single-host Slurm environment for local development and testing. Specifically, we cover `slurmctld` (the central management daemon) and `slurmd` (the compute node daemon) now and plan to include `slurmdbd` and `slurmrestd` in the future.

This guide has been tested on Ubuntu 20.04.6 LTS with the `sshd` service already running. For instructions on setting up `sshd`, please refer to the [OpenSSH server documentation](https://ubuntu.com/server/docs/openssh-server).


## Table of Contents
* [Setup a Tiny Slurm Cluster](https://github.com/JiangJiaWei1103/Slurm-101/blob/main/README.md#setup-a-tiny-slurm-cluster)
    * [MUNGE First](https://github.com/JiangJiaWei1103/Slurm-101/blob/main/README.md#munge-first)
    * [Create a Dedicated Slurm User](https://github.com/JiangJiaWei1103/Slurm-101/blob/main/README.md#create-a-dedicated-slurm-user)
    * [Make Slurm Run](https://github.com/JiangJiaWei1103/Slurm-101/blob/main/README.md#make-slurm-run)
* [Try Some Commands](https://github.com/JiangJiaWei1103/Slurm-101/blob/main/README.md#try-some-commands)
* [Use Case](https://github.com/JiangJiaWei1103/Slurm-101/blob/main/README.md#use-case)

## Setup a Tiny Slurm Cluster
### MUNGE First
> [MUNGE](https://dun.github.io/munge/) is an authentication service, allowing a process to authenticate the UID and GID of another local or remote process within a group of hosts having common users and groups.

First, we install necessary packages. 
```shell
sudo apt install munge libmunge2 libmunge-dev
```

Then, we run the following command to generate a MUNGE credential, decode and verify the encrypted token, and filter the output to show the verification result.
```shell
munge -n | unmunge | grep STATUS
```

A status of `STATUS: Success(0)` is expected and the MUNGE key is stored at `/etc/munge/munge.key`. If the key is absent, please run the following command to create one manually.
```shell
sudo /usr/sbin/create-munge-key
```

The `munged` daemon should be run as a dedicated non-privileged user. Fortunately, this user is automatically created and called "munge". We use the following commands to change the ownership of specific MUNGE-related directories and adjust permissions.
```shell
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo chmod 0755 /run/munge/
sudo chmod 0700 /etc/munge/munge.key
sudo chown -R munge: /etc/munge/munge.key
```

Finally, we make `munged` start at boot and restart the service. 
```shell
sudo systemctl enable munge
sudo systemctl restart munge
```

To check if the daemon runs as expected, we can either use `systemctl status munge` or inspect the log file under `/var/log/munge`.

I supposed I could ignore this step for the tiny setup, but it turns out this step is necessary. Still pondering why...

### Create a Dedicated Slurm User 
> The *SlurmUser* must be created as needed prior to starting Slurm and must exist on all nodes in your cluster.

Before installing and running `slurmctld` and `slurmd`, we create a dedicated Slurm system user first. This setup order may be different from the [official instructions](https://slurm.schedmd.com/quickstart_admin.html#quick_start), but we find that this can avoid some troubles of creating an user and altering folder ownership afterward.

To create a dedicated Slurm user, we run the following command. We make sure `uid` is equal to `gid` to keep our nose clean. For details about adding a system user, please refere to the section [Add a system user](https://manpages.ubuntu.com/manpages/oracular/en/man8/adduser.8.html).
```shell
# Choose an uid (we choose 152)
# A system user usually has an uid in the range of 0-999
adduser --system --uid <uid> --group --home /var/lib/slurm slurm

# Check Slurm user
cat /etc/passwd | grep <uid>
```

It's of vital importance to set correct ownership of specific Slurm-related directories to prevent access issue. Directories mentioned below will be created automatically when we start Slurm services. However, we decide to manually create them and alter the ownership beforehand.
```shell
sudo mkdir -p /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
sudo chown -R slurm: /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
```
<!-- /etc/slurm (root): Store `slurm.conf` and `slurmdbd.conf` -->
<!-- /var/run/slurm (slurm): Don't know for what -->

### Make Slurm Run
After the preparatory work is complete, we move on to the core steps.

#### Install Slurm Packages
First, we download Slurm source [here](https://www.schedmd.com/download-slurm/) and choose version `24.05.5`. We recommend to download the file to a clean directory because debian packages will be generate under this dir in the following steps. 
```shell
wget -P setup_slurm/ https://download.schedmd.com/slurm/slurm-24.05.5.tar.bz2
```

After downloading, we build debian packages following this [official guide](https://slurm.schedmd.com/quickstart_admin.html#debuild).
```shell
# Install basic Debian package build requirements
sudo apt-get update
sudo apt-get install build-essential fakeroot devscripts equivs

# (Optional) Install dependencies if missing
sudo apt install -y \
    libncurses-dev libgtk2.0-dev libpam0g-dev libperl-dev liblua5.3-dev \
    libhwloc-dev dh-exec librrd-dev libipmimonitoring-dev hdf5-helpers \
    libfreeipmi-dev libhdf5-dev man2html-base libcurl4-openssl-dev \
    libpmix-dev libhttp-parser-dev libyaml-dev libjson-c-dev \
    libjwt-dev liblz4-dev libmariadb-dev libdbus-1-dev librdkafka-dev

# Unpack the distributed tarball
tar -xaf slurm-24.05.5.tar.bz2

# cd to the directory containing the Slurm source
cd slurm-24.05.5

# Install the Slurm package dependencies
sudo mk-build-deps -i debian/control

# Build the Slurm packages
debuild -b -uc -us
```

Now, debian pakcages are built and placed under the parent directory (`setup_slurm/` in our case). We continue to install targer packages that suit our needs. As we setup a single-host tiny Slurm cluster acting as a controller and a compute node at the same time, we need to install `slurm-smd`, `slurm-smd-client` (for CLI), `slurm-smd-slurmctld`, and `slurm-smd-slurmd`. For details, please refer to [Installing Packages](https://slurm.schedmd.com/quickstart_admin.html#pkg_install).
```shell
# cd to the parent directory
cd ..

sudo dpkg -i slurm-smd_24.05.5-1_amd64.deb
sudo dpkg -i slurm-smd-client_24.05.5-1_amd64.deb
sudo dpkg -i slurm-smd-slurmctld_24.05.5-1_amd64.deb
sudo dpkg -i slurm-smd-slurmd_24.05.5-1_amd64.deb
```

#### Configure Slurm
Here comes the most tricky part, configuring Slurm. Specifically, we need to generate a correct `slurm.conf` file to make Slurm work. It's thanks to this [official configurator](https://slurm.schedmd.com/configurator.html) that we complete this mission smoothly.

Following show key-value pairs we set manually. Please leave other options untouched because default settings are sufficient to make `slurmctld` and `slurmd` run.
```
# == Cluster Name ==
ClusterName: localcluster

# == Control Machines ==
SlurmctldHost: localhost

# == Compute Machines ==
NodeName: localhost

# lscpu | egrep 'Socket|Thread|CPU\(s\)'
CPUs: 16
Sockets: 1
CoresPerSocket: 8
ThreadsPerCore: 2

# free -m
# We use available
RealMemory: 30528

# == Process Tracking ==
ProctrackType: LinuxProc

# == Event Logging ==
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
```

After filling out the form, submit it, copy the content and write the content to `/etc/slurm/slurm.conf`.

#### Start Daemons
Nex, we make `slurmctld` and `slurmd` start at boot and restart them.
```shell
# For controller
sudo systemctl enable slurmctld
sudo systemctl restart slurmctld

# For compute
sudo systemctl enable slurmd
sudo systemctl restart slurmd
```

We can again check `systemctl status <daemon>` or inspect log files `/var/log/slurm/slurmctld.log` and `/var/log/slurm/slurmd.log`.

## Try some commands
With `slurmctld` and `slurmd` running, we can now play around with some simple commands to see what will happen.

* `sinfo`: View information about Slurm nodes and partitions
```shell
root@rockwei:/etc/slurm# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      1   idle localhost
```
* `srun`: Run a parallel job on cluster managed by Slurm
```shell
root@rockwei:/etc/slurm# srun -N 1 hostname
rockwei
```

Following shows one small tip to enable job submission if the state becomes `drain`. We set the state back to `idle` as follows:
```shell
scontrol update nodename=localhost state=idle
```

<!---->
<!-- ### `slurmdbd` -->
<!-- Mainly follow  -->
<!-- https://medium.com/@hghcomphys/building-slurm-hpc-cluster-with-raspberry-pis-step-by-step-guide-ae84a58692d5 -->
<!-- 1. `dpkg -i dbd...` -->
<!-- 2. SQL server or mariadb -->
<!-- 3. Setup db -->
<!-- ``` -->
<!-- create database slurm_acct_db; -->
<!-- create user 'abaowei'@'localhost'; -->
<!-- set password for 'abaowei'@'localhost' = 'slurmdbpass'; -->
<!-- grant usage on *.* to 'abaowei'@'localhost'; -->
<!-- grant all privileges on slurm_acct_db.* to 'abaowei'@'localhost'; -->
<!-- flush privileges; -->
<!-- exit -->
<!-- ``` -->
<!-- 4. Write `/etc/slurm/slurmdbd.conf` -->
<!-- 5. chmod and chown  -->
<!-- ``` -->
<!-- sudo chmod 600 /etc/slurm/slurmdbd.conf -->
<!-- sudo chown abaowei:slurm /etc/slurm/slurmdbd.conf -->
<!-- ``` -->
<!-- 6. Setup `/usr/lib/systemd/system/slurmdbd.service` -->
<!---->
<!---->
<!-- #### Config Ref -->
<!-- * https://gist.github.com/hackprime/486759fa98bf0112aed8302a036526e3#file-slurmdbd-conf-L13 -->
<!---->
<!---->
<!-- ### `slurmrestd` -->
<!-- My setup mainly refers to  -->
<!-- https://blog.csdn.net/zrc_xiaoguo/article/details/134658813?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-134658813-blog-109687256.235^v43^pc_blog_bottom_relevance_base4&spm=1001.2101.3001.4242.1&utm_relevant_index=3 -->
<!-- (How to write `slurmrestd.service` and gen jwt auth key) -->
<!---->
<!-- and  -->
<!-- https://slurm.schedmd.com/rest_quickstart.html#quick_start -->
<!---->
<!-- Basic usage and test refers to  -->
<!-- https://slurm.schedmd.com/rest_quickstart.html#basic_usage -->
<!---->
<!-- Common command to gen and use new auth token -->
<!-- ``` -->
<!-- unset SLURM_JWT; export $(scontrol token) -->
<!-- ``` -->
<!---->
<!---->
<!---->
<!---->
<!-- Great example sources -->
<!-- * https://indico.ihep.ac.cn/event/22838/contributions/159617/attachments/79052/98526/slurm_workbench.pdf -->
<!-- * (Very inspiring) https://slurm.schedmd.com/SLUG23/REST-API-SLUG23.pdf -->
<!---->
<!-- ### Convert to OpenAPI python package -->
<!-- ``` -->
<!-- sudo slurmrestd -f /dev/null --generate-openapi-spec -s slurmdbd,slurmctld -d v0.0.41 -u slurmrestd > openapi.json -->
<!-- ``` -->
<!---->
<!-- Use docker to build without additional installation -->
<!-- ``` -->
<!--  docker run --rm -v $PWD:/local openapitools/openapi-generator-cli generate -i /local/openapi.json -g python -o /local/py_api_client -->
<!-- ``` -->
<!-- Refer to https://openapi-generator.tech/#try -->
<!---->
<!-- * https://slurm.schedmd.com/slurmrestd.html#SECTION_EXAMPLES -->
<!---->
<!---->
<!---->
<!---->
<!-- ## References -->
<!-- Those really helps me setup and debug -->
<!---->
<!-- * -->
<!---->
<!---->
<!-- ## Common Issues -->
<!-- ### slurmdbd: error: cannot find auth plugin for auth/munge  -->
<!-- My workaround is to copy `/usr/lib/x86_64-linux-gnu/slurm` to `/usr/lib/slurm` -->
<!-- I know this isn't the best -->
<!-- Maybe should simply modify env var -->
<!-- * https://stackoverflow.com/questions/78435529/slurmdbd-error-cannot-find-auth-plugin-for-auth-munge -->
<!-- * https://stackoverflow.com/questions/48410583/slurm-standalone-system-ubuntu-16-04-3-compiled-not-working-authentication -->
<!-- ### error: Database settings not recommended values: innodb_buffer_pool_size innodb_lock_wait_timeout -->
<!-- Manually login to mysql `sudo mysql -u root` and -->
<!-- `SET GLOBAL innodb_buffer_pool_size=34359738368;` 32G (50-80% of the server RAM)  -->
<!---->
<!-- `SET GLOBAL innodb_lock_wait_timeout=900` -->
<!-- * https://slurm.schedmd.com/accounting.html#slurm-accounting-configuration-before-build -->
<!--     * official setup recommended -->
<!-- * https://wiki.fysik.dtu.dk/Niflheim_system/Slurm_database/#mysql-configuration -->
<!-- ### Job for slurmdbd.service failed because a timeout was exceeded. -->
<!-- * slurmdbd.service forking to simple  -->
<!-- ### slurmdbd: error: mysql_db_insert_ret_id: We should have gotten a new id: Table 'slurm_acct_db.localcluster_job_table' doesn't exist -->
<!-- Manually register the cluster -->
<!-- * https://github.com/giovtorres/slurm-docker-cluster/issues/9 -->
<!-- * https://github.com/giovtorres/slurm-docker-cluster/blob/main/register_cluster.sh -->
<!-- ### _slurm_rpc_allocate_resources: Invalid account or account/partition combination specified -->
<!-- Good start to hint me to add user -->
<!-- https://support.schedmd.com/show_bug.cgi?id=6306 -->
<!---->
<!-- Very good explaination -->
<!-- > If it contains limits, qos, wckeys, safe, or associations, then that user account must have an association in the accounting database. -->
<!-- https://slurm-dev.schedmd.narkive.com/OVnKwZO3/invalid-account-or-account-partition#post2 -->
<!---->
<!-- Here is the adding user example https://slurm.schedmd.com/sacctmgr.html#SECTION_EXAMPLES -->
<!-- ``` -->
<!-- # We've created before to solve the prev issue -->
<!-- # sacctmgr create cluster localcluster  -->
<!---->
<!-- sacctmgr create account name=flyte  -->
<!-- sacctmgr create user name=abaowei cluster=localcluster account=flyte -->
<!-- ``` -->
<!---->
<!---->
<!-- ### `slurmrestd` any curl encounters Authentication failure -->
<!-- 1. Gen JWT key -->
<!-- 2. Enable auth/jwt in slurm.conf -->
<!-- 3.  -->
<!---->
<!---->
<!-- Maybe helpful ref -->
<!-- https://lists.schedmd.com/pipermail/slurm-users/2022-March/008497.html -->
<!-- https://github.com/aws/aws-parallelcluster/issues/3865 -->
<!---->
<!---->
<!-- ## Paths -->
<!-- * `/var/lib/slurm`: Have some info about `slurmctld` and `slurmd`, but don't know the details -->
<!-- * `~/dev/Slurm-101`: All deb files for installing packages.  -->
<!---->
<!---->
<!---->
<!---->
<!-- ## Result -->
<!-- Now, we're able to interact with slurm through python interface, as shown below: -->
<!---->
<!-- ![Screenshot 2024-12-15 at 11 42 11 PM](https://github.com/user-attachments/assets/071018ee-f1ce-415b-a733-fe79e0a7245d) -->


## Todos
* [ ] Use `cgroup.conf`?
* [ ] Use docker-compose to start all services at once 
* [ ] For `slurmrestd`, use port forward and use postman for quick test


## Use Case
### Flyte Slurm Agent
See https://github.com/flyteorg/flytekit/pull/3005. Stay tuned!


## References
* [Quick Start Administrator Guide](https://www.schedmd.com/slurm/installation-tutorial/)
* [SergioMEV/slurm-for-dummies](https://github.com/SergioMEV/slurm-for-dummies)
* [How to quickly set up Slurm on Ubuntu 20.04 for single node workload scheduling.](https://drtailor.medium.com/how-to-setup-slurm-on-ubuntu-20-04-for-single-node-work-scheduling-6cc909574365)
* [Building a Slurm HPC Cluster with Raspberry Pi’s: Step-by-Step Guide](https://medium.com/@hghcomphys/building-slurm-hpc-cluster-with-raspberry-pis-step-by-step-guide-ae84a58692d5)
* [Setup Slurm cluster for HPC](https://medium.com/@satishdotpatel/setup-slurm-cluster-for-hpc-bfa73364706a)
* [Setup Slurm-web for Slurm HPC Clusters](https://medium.com/@satishdotpatel/setup-slurm-web-for-slurm-hpc-clusters-13a9873094a1)
### Chinese
* [Slurm 20.02.3 CentOS 7.6 slurm slurmdbd 安装教程 No. 5-1](https://blog.csdn.net/xuecangqiuye/article/details/109687256)
* [Slurm setup 專欄](https://blog.csdn.net/xuecangqiuye/category_10212500.html)
* [slurm–核算和资源限制](https://yaohuablog.com/zh/slurm-accounting-and-resource-limits)
* [SLURM超算集群资源管理服务的安装和配置-基于slurm22.05.9和centos9stream，配置slurmdbd作为账户信息存储服务](https://blog.csdn.net/zrc_xiaoguo/article/details/134634440?spm=1001.2014.3001.5502)
* [SLURM资源调度管理系统REST API服务配置，基于slurm22.05.9，centos9stream默认版本](https://blog.csdn.net/zrc_xiaoguo/article/details/134658813?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-134658813-blog-109687256.235%5Ev43%5Epc_blog_bottom_relevance_base4&spm=1001.2101.3001.4242.1&utm_relevant_index=3)








