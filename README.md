# DataHUB-cluster

## Description

This [Ansible](#ansible) library deploys a cluster of hosts for a high-availability Gitlab instance. The hosts are co-ordinated via [Corosync/Pacemaker](https://clusterlabs.org/) and storage is synced via [DRBD](https://linbit.com/drbd/). In addition to the DataHUB hosts, there are playbooks and roles for deploying supporting services, specifically an [HAProxy](https://www.haproxy.com/) external proxy, and [Keycloak](https://www.keycloak.org/) for authentication.

For backups, there are tasks to co-ordinate simple gitlab backup on the DataHub hosts, as well as more complex syncing of repositories and data between the Ansible controller and the hosts. For DataHUBs making use of S3 storage, there are tasks to sync buckets between the controller and one or more S3 regions.

For post-deployment management, there are tasks to carry out upgrades of the host and the applications, as well as configuration changes. There is a playbook to schedule tasks with systemd timers for after-hours and/or regular scheduling of upgrades.

After a DataHUB cluster has been deployed, it is sometimes necessary to rebuild and re-integrate a cluster host. There is a short guide to re-installing gitlab on a host in an existing DataHUB cluster, as well as a playbook to rebuild and re-integrate a host. 

Note: This library was developed to support the DataPLANT DataHUB research data repository. As such, the [main Gitlab configuration file](roles/gitlab-cluster-host/templates/gitlab.rb.j2) and some [ruby configuration files](roles/gitlab-cluster-host/tasks/apply_gitlab_patches.yml) are modified to facilitate the transfer of large files to and from the DataHUB cluster.

### Structure of the library and how to interact

The cluster and related services are deployed and managed using the top-level playbooks. These are specific to the kind of host being targeted:

- [DataHUB (Gitlab) cluster host](gitlab_cluster_host.yml)
- [HAProxy external proxy host](proxies.yml)
- [Keycloak host](keycloaks.yml)

There are two additional playbooks, one to [redeploy a host](rebuild_gitlab_cluster_host.yml) into an existing cluster, and one to [schedule playbooks](controller_systemd_playbook_triggers.yml) as systemd timers.

Playbooks are executed with the ansible-playbook executable. In addition to the playbook, you will need to also specify an [inventory environment](#environments) (_which hosts_ to work on) and [tags](#roles-and-tags) (_which tasks_ to apply):

>```ansible-playbook [playbook].yml -i environment/[environment_directory]/ --tags [applicable tag(s)]```

For example:

>```ansible-playbook gitlab_cluster_host.yml -i environments/testing/ --tags deploy,config-secrets-update```

Before starting, you will also need to configure [SSH access from the controller to the hosts](https://docs.ansible.com/projects/ansible/latest/inventory_guide/connection_details.html) and establish [a vault key](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html) to encrypt authentication strings in the environment variables. For convenience, the keys can be placed in a top level ```keys/``` directory, which will be ignored by git.

#### Environments

The Ansible inventory is the listing of hosts that will be managed, along with host and host group specific variables. [Ansible inventories](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html) can be [structured differently](https://docs.ansible.com/projects/ansible/latest/tips_tricks/sample_setup.html). In this library, the inventories are held in an ```environments``` directory to facilitate managing multiple clusters, which is useful for running parallel ```test``` and ```production``` clusters. Hosts and host groups are specified in the ```hosts.yml``` file, and all host and host group variables are located in a folder structure. The library comes with this folder structure and example variable files (with the extension```.example``` ) for each host and host group. 

In the following sections, the variables that need to be filled in for each function will be described.

Note: If preferred, it is possible to collapse the inventory to a single inventory file. However the host group names must be preserved, as the groups are referenced by name in some playbooks and tasks.


#### Roles and tags

This library makes use of the Ansible tag system. Each task is associated to: 
- a tag for just that task or small task set
- a tag for a set of related tasks
- a tag with the role name

For example, the task which queries the repository and the host gitlab version looks like this:

```
- name: Gitlab version check
  ansible.builtin.import_tasks:
    file: version_gitlab.yml
  tags:
    - version-gitlab
    - upgrade-gitlab
    - gitlab-cluster-host
    - never
 ```
It can be called by itself, as part of the set of tasks that upgrades the Gitlab version on all cluster hosts, or as part of the set of all tasks in the role. The ```-never``` ensures it is not applied unless one of the other tags is present.

The majority of tasks can be found under different roles in the ```roles/``` directory. To see an index of tasks and associated tags for task or task set, see the ```/roles/[role name]/tasks/main.yml``` file. 

So you don't have to go rummaging in the roles to know which tags to use, below is a description of how to use tags to carry out each kind of deploy, reconfiguration or maintenance task.


## DataHUB (Gitlab) cluster host

### Deploy

#### Hosts, host variables

For the gitlab cluster you will need to provision a minimum of 3 hosts. These hosts are given hostnames in the environment [```hosts.yml```](environments/testing/hosts.yml) host group ```gitlab_cluster```. In the environment directory ```host_vars``` there must be a host variables file for each [cluster host](environments/testing/host_vars/gitlab-host.yml.example). In this file is specified the IP address of the host and an identifier for the local volume(s) that will be used for replication of gitlab data. The storage volume(s) must be available in the host as unformatted, unmounted volumes. 

Note: For simplicity, the library assumes the hosts are running Ubuntu 24.04. As of Gitlab v19.4.0-ce.0, Ubuntu 26.04 is supported, however it has not yet been tested if _all_ components in this library are supported for Ubuntu 26.04. 


#### Gitlab cluster variables

The source of the variables used to populate the Gitlab configuration file is the environment ```group_vars``` [```gitlab_cluster.yml```](environments/testing/group_vars/gitlab_cluster.yml.example) file . Before deploying, the variables that **must** be populated are:
- ````gitlab_shared_ip````, this is a shared, fixed IP that the primary Gitlab host will bind. It must be pre-configured before deployment.
- ```gitlab_initial_root_pw```, the root password used to initialise the gitlab instance. It is strongly recommended to use an [Ansible vault string](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_encrypting_content.html#creating-encrypted-variables).

For the remaining variables, these are not used for deployment and are applied by [reconfiguring](#modifying-gitlab-configuration) on the established cluster.

To deploy the cluster, run the ```gitlab_cluster_host.yml``` playbook with the deploy tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags deploy```


### Backup

Backups incorporate the standard built-in gitlab backup, as well as additional tasks for repository and S3 buckets sync that is necessary for instances that will handle large files. 

#### Built-in Gitlab backup
Gitlab has a good [built-in backup](https://docs.gitlab.com/administration/backup_restore/backup_gitlab/) system that will create a single archive of repositories and DB, as well as many of the other components of a Gitlab instance. The backup is created in the ```/var/opt/gitlab/backups``` with an option to [sync to an S3 bucket](environments/testing/group_vars/gitlab_cluster.yml.example).

To initiate a Gitlab backup, run the ```gitlab_cluster_host.yml``` playbook with the ```backup-gitlab``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags backup-gitlab```

The default behaviour for Gitlab is to trigger a backup for each version upgrade. **This behaviour has been disabled in the DataHUB cluster**. Backup must be initiated with the playbook manually or via scheduling.

For Gitlab instances that will handle large files, the standard backup process will take a very long time to create the backup archive file. For these instances, it is strongly recommended to follow the procedure below for additional repository and S3 bucket sync instead of the built-in backup alone. 

#### Repository and S3 bucket sync

For instances that will handle large files, it is strongly recommended to use [S3 object storage](https://docs.gitlab.com/administration/object_storage/), however, this storage is not captured by the built-in Gitlab backup. In addition, the repositories and pipeline artifacts store may contain large files that take a long time to be collected by the built-in backup. To manage backups in this context, a sync mechanism has been built in to this library that uses the Ansible controller to sync S3 buckets and/or repositories to a local backup volume and, if desired, a secondary S3 destination.

The following variables **must** be set in the environment [```group_vars/gitlab_cluster.yml``` ](environments/testing/group_vars/gitlab_cluster.yml.example) file:

- ```backup_skip```, this is a list of all data sources to skip when creating a backup archive using the standard gitlab backup service. An example string is given in the example file that will capture _only_ the Gitlab DB. Adjust to suit your setup.

- ```device_for_backup_sync```, volume path or UUID for backup sync volume
- ```fstype_for_backup_sync```, filesystem type of backup sync volume
- ```mount_for_backup_sync```, mount path for backup sync volume

##### S3 sync to Ansible controller

To sync S3 buckets to the Ansible controller volume specified above, run the ```gitlab_cluster_host.yml``` playbook with the ```backup-sync-s3-to-local``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags backup-sync-s3-to-local```

Note: no additional variables are required, as Ansible already has the object store credentials.

##### Repository sync

The variable ```repository_backup_name``` in the environment [```group_vars/gitlab_cluster.yml``` ](environments/testing/group_vars/gitlab_cluster.yml.example) file must have a value. This will be used to create and refer to the directory on the mounted sync volume above. 

To sync the Gitlab repositories to the Ansible controller volume specified above, run the ```gitlab_cluster_host.yml``` playbook with the ```backup-sync-repos-to-local``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags backup-sync-repos-to-local```

##### Sync to secondary S3

To sync to a secondary S3 destination, the details must be entered in the environment [```group_vars/gitlab_cluster.yml``` ](environments/testing/group_vars/gitlab_cluster.yml.example) file:
- ```s3_secondary_endpoint```
- ```s3_secondary_access_key``` 
- ```s3_secondary_secret_key```, it is strongly recommended to use an [Ansible vault string](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_encrypting_content.html#creating-encrypted-variables).

To sync the local controller volume to a secondary S3 destination (incuding repositories if applicable), run the ```gitlab_cluster_host.yml``` playbook with the ```backup-sync-local-to-secondary-s3``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags backup-sync-local-to-secondary-s3```

#### Full backup and sync

To carry out all of the above backup tasks as one task set, fill in the applicable variables from the sections above, and run the ```gitlab_cluster_host.yml``` playbook with the ```backup-and-s3-sync``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags backup-and-s3-sync```

Note: If the group variable ```repository_backup_name``` is blank, repository sync will be skipped. If ```s3_secondary_endpoint``` is blank, backup to a secondary S3 destination will be skipped.

### Modifying Gitlab configuration

Changes are made to the Gitlab configuration via the central configuration file ```/etc/gitlab/gitlab.rb```. This can be done by modifying values in the environment [```group_vars/gitlab_cluster.yml``` ](environments/testing/group_vars/gitlab_cluster.yml.example) file or, for values not inserted into the template, by modifying the [gitlab.rb template](roles/gitlab-cluster-host/templates/gitlab.rb.j2) itself. The changed configuration is applied by running the ```reconfigure``` of the Gitlab application. To regenerate and sync the updated ```gitlab.rb``` file to Gitlab cluster hosts and trigger a reconfigure by the Gitlab applications on the cluster, run the ```gitlab_cluster_host.yml``` playbook with the ```config-secrets-update``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags config-secrets-update```


### Upgrading

By default the library will choose the latest version of Gitlab when upgrading. To choose a Gitlab version, specify it in the environment [```group_vars/gitlab_cluster.yml``` ](environments/testing/group_vars/gitlab_cluster.yml.example) file ```gitlab_version``` variable. 

To upgrade the Gitlab version on the cluster, run the ```gitlab_cluster_host.yml``` playbook with the ```upgrade-gitlab``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags upgrade-gitlab```

#### Apply Gitlab patches

To facilitate large file transfers to the Gitlab, we make some patches to Gitlab configuration files. When a new Gitlab version is installed, these are overridden. After each upgrade, the patches need to be re-applied and the primary Gitlab application reloaded. To re-apply the patches and reload Gitlab, run the ```gitlab_cluster_host.yml``` playbook with the ```patch-gitlab-after-upgrade``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags patch-gitlab-after-upgrade```

Both tags can be applied in the same run. To upgrade Gitlab on the cluster and then apply patches:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags upgrade-gitlab,patch-gitlab-after-upgrade```


### Rebuild, reinstall and reintegrate a cluster host

#### Gitlab application re-install

A Gitlab cluster host, when running as the primary active host, uses a mix of local data (generic application data) and data on the shared replicated volume (instance specific data). It can occur that the local application data of a Gitlab cluster host needs to be re-installed.

For this there is no role or playbook, but the process is relatively simple. It will require SSH login to the host where Gitlab is to be re-installed (referred to as _affected host_).
<details>
<summary>Gitlab re-install procedure:</summary>

1. On any Gitlab cluster host, take the affected host out of the cluster: 

>```sudo pcs cluster stop [affected host]```

2. On the affected host, uninstall and remove local Gitlab files 
> ```sudo gitlab-ctl uninstall```

>```sudo apt remove gitlab-ce```

>```sudo rm -r /etc/gitlab /var/opt/gitlab/ /var/log/gitlab/ /opt/gitlab /usr/lib/systemd/system/gitlab-runsvdir.service```

3. On the Ansible controller, run the ```gitlab_cluster_host.yml``` playbook with the ```install-gitlab, config-sync``` and ```secrets-sync``` tags:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags install-gitlab,config-sync,secrets-sync, --limit [affected host]```

4. On the affected host, swap the shared cluster ```gitlab-secrets.json``` file with the local version

>```sudo mv /etc/gitlab/gitlab-secrets.json /etc/gitlab/gitlab-secrets.json.cluster && sudo mv /etc/gitlab/gitlab-secrets.json.orig /etc/gitlab/gitlab-secrets.json```

5. On the affected host, run the first local reconfigure

>```sudo gitlab-ctl reconfigure```

6. On the affected host, stop the Gitlab service

> ```sudo systemctl stop gitlab-runsvdir```

7. On the affected host, swap back the ```gitlab-secrets.json``` file to the shared cluster version

>```sudo mv /etc/gitlab/gitlab-secrets.json /etc/gitlab/gitlab-secrets.json.orig && sudo mv /etc/gitlab/gitlab-secrets.json.cluster /etc/gitlab/gitlab-secrets.json```

8. On the affected host, rejoin the cluster

> ```sudo pcs cluster start [affected host] --request-timeout=90```

</details>

#### Gitlab host rebuild

It is possible that a Gitlab cluster host suffers an unrecoverable failure. In this case it is possible to use the existing configuration to (re-)build a new host and integrate it into the existing cluster. This host will be referred to as the _affected host_.

If the affected host will use a new local IP address, then this needs to be updated in the environment [host_vars](environments/testing/host_vars/gitlab-host.yml.example) file for the affected host. To sync the new hostname and IP to all hosts, run the ```gitlab_cluster_host.yml``` playbook with the ```hostnames``` tag:

>```ansible-playbook gitlab_cluster_host.yml -i environments/[environment name] --tags hostnames``` 

Note: This may need to also be run for proxies and keycloak if applicable.

The affected host needs to be entered into the host group ```gitlab_cluster_hosts_to_rebuild``` in the environment [hosts](environments/testing/hosts.yml) file. Note, the affected host should appear in _both_ the ```gitlab_cluster``` and ```gitlab_cluster_hosts_to_rebuild``` host groups.

Then, run the the ```rebuild_gitlab_cluster_host.yml``` playbook with the ```rebuild-host``` tag:

>```ansible-playbook rebuild_gitlab_cluster_host.yml -i environments/[environment name] --tags rebuild-host```

The re-integrated host will now begin to replicate the shared volume. This may take some time depending on the volume size. 

## HAProxy proxy

The HAProxy host can be based on a wide-range of host systems. Unless you are expecting very high traffic, the host can be small, i.e. a host with 4GB Memory and 2xCPU will handle a high volume of traffic. The host OS can be a wide range of Linux distributions using package managers apt(Debian/Ubuntu), yum/DNF (RHEL lineage), or zypper (openSUSE lineage). 

### Deploy

To deploy an HAProxy host, add a hostname under the host group ```proxies``` in the environment [hosts](environments/testing/hosts.yml) file. In a corresponding environment [host_vars](environments/testing/external-proxy.yml.example) file the following variables **must** be specified:
- ```ansible_host```, the local IP of the host, reachable from the controller.

To deploy the haproxy proxy host, run the ```proxies.yml``` playbook with the deploy tag:

>```ansible-playbook proxies.yml -i environments/[environment] --tags deploy```


### Modifying configuration

The HAProxy configuration, certificates, and a user-agent blocklist can be updated. These are specified in the ```host_vars``` entry for the proxy: 

- ```haproxy_config_file```, the haproxy.cfg file to be applied to the host. This is not templated and must be [prepared manually](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/), then placed in the ```files/``` directory of the haproxy role. It is sufficient to use a minimal placeholder file for deployment, as the config can be updated with a maintenance task.
- ```haproxy_certs_dir```, certificates specified in the configuration. The directory should be located in the ```files/``` directory of the haproxy role.
- ```haproxy_agents_blocklist```, list of user blocked user agents. An extensive blocklist that blocks most crawlers is _already_ in the ```files/``` directory of the haproxy role. Make sure to remove or edit this list if some crawler activity is desired (e.g. search crawlers).

After modifying any of the above files, run the ```proxies.yml``` playbook with the ```update-config-certs``` tag:

>```ansible-playbook proxies.yml -i environments/[environment] --tags update-config-certs```
 
### Upgrading

The HAProxy application will be upgraded as part of normal package updates. This can be applied from the ```base_os``` role, with the ```update_os``` tag:

>```ansible-playbook proxies.yml -i environments/[environment] --tags update_os```


## Keycloak

As for the proxy, the Keycloak host can be based on a wide-range of host systems. Unless you are expecting very high traffic, the host can be small, i.e. a host with 4GB Memory and 2xCPU will handle a high volume of traffic. The host OS can be a wide range of Linux distributions using package managers apt(Debian/Ubuntu), yum/DNF (RHEL lineage), or zypper (openSUSE lineage).

### Deploy

To deploy a Keycloak host, add a hostname under the host group ```keycloaks``` in the environment [hosts](environments/testing/hosts.yml) file. In a corresponding [host_vars](environments/testing/keycloak.yml.example) file the following variables **must** be specified:
- ```ansible_host```, the local IP of the host, reachable from the controller.
- ```keycloak_hostname```, hostname and context path (if applicable) where the keycloak instance will be served.
- ```keycloak_initial_admin_name```, username for the admin user.
- ```keycloak_initial_admin_pw```, password for the admin user. It is strongly recommended to use an [Ansible vault string](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_encrypting_content.html#creating-encrypted-variables).
- ```keycloak_postgres_pw```, password for the keycloak DB. As above, it is strongly advised to use an Ansible vault string.
 
Note: After deployment of the keycloak, both the Keycloak and the Gitlab instance will need further configuration to enable authentication. This is outside the scope of this document, a good starting point would be the [Gitlab OIDC docs](https://docs.gitlab.com/integration/openid_connect_provider/).

To deploy the Keycloak host, run the ```keycloaks.yml``` playbook with the ```deploy``` tag:

>```ansible-playbook keycloaks.yml -i environments/[environment] --tags deploy```

Note: By default, the library will pull the latest version. To choose a Keycloak version, enter a value into the host variable ```keycloak_version```.

### Modifying configuration

This library does not include post-deployment configuration. This can be best carried out via the Keycloak admin UI, using the admin credentials created during deployment.

### Upgrading

Keycloak is not installed or upgraded through a package manager, rather it needs to be pulled from the repository and unpacked. To trigger these tasks for an upgrade, run the ```keycloaks.yml``` playbook with the ```upgrade_keycloak``` tag:

>```ansible-playbook keycloaks.yml -i environments/[environment] --tags upgrade_keycloak```


### Backup

To trigger a Keycloak DB backup, run the ```keycloaks.yml``` playbook with the ```backup_keycloak``` tag:

>```ansible-playbook keycloaks.yml -i environments/[environment] --tags backup_keycloak```

This will drop a backup with the naming format ```[date]-[keycloak version]``` in the directory  ```/srv/keycloak/backups/``` on the Keycloak host.

## Scheduling playbooks with systemd timers

Many of the maintenance operations are best run out of hours to avoid disruption to DataHUB instance users. To facilitate this, a role has been created to place systemd timers that trigger services to run our ansible playbooks. 

The timer and service to be created must be specified in the [environment group_vars all.yml](environments/testing/group_vars/all.yml.example) file, in the ```services_timers``` dictionary. A full set of example timer definitions is already in the example file. The dictionary entry starts with a name, which will used to name of the systemd service and timers. In each entry, specify:
- ```playbook```, the name of the playbook that will be executed
- ```[optional] tags```, any playbook tags that should be added to the playbook run. This tag is optional, but, in this library, at least one tag is requiredfor any tasks to be carried out.
- ```[optional] hosts```, specify any hosts here to restrict the playbook to these hosts.
- ```calendar_entry```, a [systemd format calendar entry](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html#Calendar%20Events).

To set up, or to modify a timer and associated service, run the ```controller_systemd_playbook_triggers.yml``` playbook with the ```systemd-playbook-triggers``` tag:

>```ansible-playbook controller_systemd_playbook_triggers.yml -i environments/[environment] --tags systemd-playbook-triggers```

Note, removing a timer from the dictionary will not cause it to be removed from the system. Stopping and/or removing a timer will need to be done manually. 

Timers can be stopped and/or disabled by: 
>```sudo systemctl stop [timer name].timer```
>```sudo systemctl disable [timer name].timer```

To completely remove a timer and service, remove the following files: 
>```/etc/systemd/system/[timer/service name].timer```
>```/etc/systemd/system/[timer/service name].service```

## Ansible
The heart of the project is the [Ansible](https://docs.ansible.com) library. Ansible is a scripting language tailored to the deployment and management of computing resources. It is intended to be run from a central control node, which then deploys python scripts to carry out operations on remote nodes. Ansible is a **declarative** language. The administrator declares the desired state of the systems being managed, and the Ansible runtime ensures the systems being managed match this declaration. Actions are therefore only performed _if_ it is required to bring the system to the declared state.

### Prepare the Ansible controller

1. Create a group for all users who will have access to the Ansible library. Add users to this group as required.
	``` 
	groupadd ansible-admins
	gpasswd -a [user] ansible-admins 
	``` 
2. Create a location for the Ansible library in ```/srv/ansible```.
	```
	mkdir /srv/ansible
	chgrp ansible-admins /srv/ansible
	chmod 2770 /srv/ansible
	``` 
3. Install Ansible in a python environment
	+ Install python pip and venv 
	```
	apt-get install python3-pip python3-venv
	```
	+ Creat a python virtual environment in the shared /srv/ansible space
	```
	 python3 -m venv /srv/ansible/env-datahub-cluster
	```
	+ Activate the virtual environment
	```
	source /srv/ansible/env-datahub-cluster/bin/activate
	``` 
	
	+ Install Ansible python libraries via ```pip```
	```
	pip install ansible
	```
4. Clone the DataHUB cluster project 
	```
	git clone https://github.com/nfdi4plants/DataHUB-cluster.git /srv/ansible/datahub-cluster
	```
	or, if ssh is preferred
	```
	git clone git@github.com:nfdi4plants/DataHUB-cluster.git /srv/ansible/datahub-cluster
	```
5. Place initial key files
	+ Copy the Ansible controller private key to ```keys/```. Ideally, this is the key that was used to initialize the other VMs. If it is created at this step, ensure the public key has been added to the VMs that will be managed by the Ansible controller.
	
	+ Create a vault key
	```
	openssl rand -base64 10 > /srv/ansible/datahub-cluster/keys/vault_key