# Slurm-101


## [WIP] Don't Read It!!!!
I start learning slurm for about 1 week, so needing some time to digest and organize the note.

Setup a single-host slurm cluster for local dev.

For unbuntu 20.04


* Good resources but too vague for me 
    * https://www.schedmd.com/slurm/installation-tutorial/
    * [Quick Start Administrator Guide](https://slurm.schedmd.com/quickstart_admin.html)
        * Official guide



## Steps
### Munge
* https://github.com/SergioMEV/slurm-for-dummies?tab=readme-ov-file
    * No `slurmrestd`

### A Dedicated slurm user
Mine is abaowei in group slurm
But most tutorials recommend just create a slurm:slurm user in group
* https://blog.csdn.net/xuecangqiuye/article/details/109687256
    * This article is only used to prove that it creates a dedicated slurm user, I don't refer to any other settings.

It's important to correctly setup dir ownership using `chown`
I don't setup in the beginning, which is troublesome for me during the setup process.



### `slurmdbd`
Mainly follow 
https://medium.com/@hghcomphys/building-slurm-hpc-cluster-with-raspberry-pis-step-by-step-guide-ae84a58692d5
1. `dpkg -i dbd...`
2. SQL server or mariadb
3. Setup db
```
create database slurm_acct_db;
create user 'abaowei'@'localhost';
set password for 'abaowei'@'localhost' = 'slurmdbpass';
grant usage on *.* to 'abaowei'@'localhost';
grant all privileges on slurm_acct_db.* to 'abaowei'@'localhost';
flush privileges;
exit
```
4. Write `/etc/slurm/slurmdbd.conf`
5. chmod and chown 
```
sudo chmod 600 /etc/slurm/slurmdbd.conf
sudo chown abaowei:slurm /etc/slurm/slurmdbd.conf
```
6. Setup `/usr/lib/systemd/system/slurmdbd.service`


#### Config Ref
* https://gist.github.com/hackprime/486759fa98bf0112aed8302a036526e3#file-slurmdbd-conf-L13


### `slurmrestd`
My setup mainly refers to 
https://blog.csdn.net/zrc_xiaoguo/article/details/134658813?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-134658813-blog-109687256.235^v43^pc_blog_bottom_relevance_base4&spm=1001.2101.3001.4242.1&utm_relevant_index=3
(How to write `slurmrestd.service` and gen jwt auth key)

and 
https://slurm.schedmd.com/rest_quickstart.html#quick_start

Basic usage and test refers to 
https://slurm.schedmd.com/rest_quickstart.html#basic_usage

Common command to gen and use new auth token
```
unset SLURM_JWT; export $(scontrol token)
```




Great example sources
* https://indico.ihep.ac.cn/event/22838/contributions/159617/attachments/79052/98526/slurm_workbench.pdf
* (Very inspiring) https://slurm.schedmd.com/SLUG23/REST-API-SLUG23.pdf

### Convert to OpenAPI python package
```
sudo slurmrestd -f /dev/null --generate-openapi-spec -s slurmdbd,slurmctld -d v0.0.41 -u slurmrestd > openapi.json
```

Use docker to build without additional installation
```
 docker run --rm -v $PWD:/local openapitools/openapi-generator-cli generate -i /local/openapi.json -g python -o /local/py_api_client
```
Refer to https://openapi-generator.tech/#try

* https://slurm.schedmd.com/slurmrestd.html#SECTION_EXAMPLES




## References
Those really helps me setup and debug

*


## Common Issues
### slurmdbd: error: cannot find auth plugin for auth/munge 
My workaround is to copy `/usr/lib/x86_64-linux-gnu/slurm` to `/usr/lib/slurm`
I know this isn't the best
Maybe should simply modify env var
* https://stackoverflow.com/questions/78435529/slurmdbd-error-cannot-find-auth-plugin-for-auth-munge
* https://stackoverflow.com/questions/48410583/slurm-standalone-system-ubuntu-16-04-3-compiled-not-working-authentication
### error: Database settings not recommended values: innodb_buffer_pool_size innodb_lock_wait_timeout
Manually login to mysql `sudo mysql -u root` and
`SET GLOBAL innodb_buffer_pool_size=34359738368;` 32G (50-80% of the server RAM) 

`SET GLOBAL innodb_lock_wait_timeout=900`
* https://slurm.schedmd.com/accounting.html#slurm-accounting-configuration-before-build
    * official setup recommended
* https://wiki.fysik.dtu.dk/Niflheim_system/Slurm_database/#mysql-configuration
### Job for slurmdbd.service failed because a timeout was exceeded.
* slurmdbd.service forking to simple 
### slurmdbd: error: mysql_db_insert_ret_id: We should have gotten a new id: Table 'slurm_acct_db.localcluster_job_table' doesn't exist
Manually register the cluster
* https://github.com/giovtorres/slurm-docker-cluster/issues/9
* https://github.com/giovtorres/slurm-docker-cluster/blob/main/register_cluster.sh
### _slurm_rpc_allocate_resources: Invalid account or account/partition combination specified
Good start to hint me to add user
https://support.schedmd.com/show_bug.cgi?id=6306

Very good explaination
> If it contains limits, qos, wckeys, safe, or associations, then that user account must have an association in the accounting database.
https://slurm-dev.schedmd.narkive.com/OVnKwZO3/invalid-account-or-account-partition#post2

Here is the adding user example https://slurm.schedmd.com/sacctmgr.html#SECTION_EXAMPLES
```
# We've created before to solve the prev issue
# sacctmgr create cluster localcluster 

sacctmgr create account name=flyte 
sacctmgr create user name=abaowei cluster=localcluster account=flyte
```


### `slurmrestd` any curl encounters Authentication failure
1. Gen JWT key
2. Enable auth/jwt in slurm.conf
3. 


Maybe helpful ref
https://lists.schedmd.com/pipermail/slurm-users/2022-March/008497.html
https://github.com/aws/aws-parallelcluster/issues/3865


## Result
Now, we're able to interact with slurm through python interface, as shown below:

![Screenshot 2024-12-15 at 11 42 11 PM](https://github.com/user-attachments/assets/071018ee-f1ce-415b-a733-fe79e0a7245d)

## Todos
* [ ] Why to use `cgroup.conf`
* [ ] Port forward and use postman for quick test
* [ ] Use docker-compose to start all services at once 



## Other References
* slurm setup 小專欄https://blog.csdn.net/xuecangqiuye/category_10212500.html







