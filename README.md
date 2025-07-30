# DataHUB cluster

The DataHUB cluster project is an Ansible library for deploying and managing a cloud-based gitlab cluster. 

The DataHUB cluster project underpins the NFDI4Plants DataHUB federated node at the University of Tübingen. 

The project includes:
- An Ansible library to deploy and manage a:
	+ **Gitlab cluster**
	+ **HAproxy reverse proxy** 
	+ **Keycloak identity management** (optional)
- Templated configuration files  to tailor the deployment to your environment.
- Customized configuration for handling large files in and out of Gitlab. 

This project is intended to work out-of-the-box, with only the instance specific details required (e.g. IP addresses, volume names). **Therefore, this is an opinionated deployment**. Priority has been given to stability and predictability, rather than accomodating a range of architectures and operating systems. Development towards a more generic implementation is welcome.


## Get started

### Prepare the cloud resources 
The process of provisioning the cloud resources will depend on your cloud provider. See below for [a brief summary of the required cloud resources](#requirements).


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
	 python3 -m venv /srv/ansible/dataplant-datahub-env
	```
	+ Activate the virtual environment
	```
	source /srv/ansible/dataplant-datahub-env/bin/activate
	``` 
	
	+ Install Ansible python libraries via ```pip```
	```
	pip install ansible
	```
4. Clone the DataHUB cluster project 
	```
	git clone .... /srv/ansible/datahub-cluster
	```
5. Place initial key files
	+ Copy the Ansible controller private key to ```keys/```. Ideally, this is the key that was used to initialize the other VMs. If it is created at this step, ensure the public key has been added to the VMs that will be managed by the Ansible controller.
	
	+ Create a vault key
	```
	openssl rand -base64 10 > /srv/ansible/datahub-cluster/keys/vault_key
	```

### Populate the variables
- All of the variables that describe the environment are located in the ```environments``` directory. 
- At the top level is the ```hosts.yml``` file, which lists each host and indicates if the host is a member of a group. 
- Variables specific to each host are located in ```environments/host_vars/[host name].yml```.
- Variables that apply to all members of a group are located in the directory ```environments/group_vars/[group name].yml```. This includes ```environments/group_vars/all.yml``` which holds shared variables for all hosts.
- For each of these variable stores, a blank template is provided. For each host and group, make a copy of the appropriate template and follow the descriptions and examples to fill in the required variables.
- When entering passwords and keys, it is recommended to store these as encrypted strings using ```ansible-vault```. This protects the keys in the case of accidental publishing of the variable files.
	+ Note, copying and pasting ansible vault output into the variable files can introduce YAML formatting errors that are hard to find. Sending the output directly to the variable file ensures the correct formatting is used:
	```
	ansible-vault encrypt_string "password to encrypt" --name=my_password >> environments/group_vars/my_group.yml
	```

### Deploy!

Once the variables are entered, the deployment playbooks can be run.

#### HAProxy reverse proxy
- Run the deployment 
	```	
	ansible-playbook haproxy_01_deploy.yml -i environments/
	```

- Note, if an HAProxy configuration file and SSL certificates are already prepared, these can be added to the ```files/``` directory as appropriate and will be included in the original deployment. More info on deploying the configuration and certificates is included with the [HAProxy management playbooks](#HAProxy-management).

#### Gitlab cluster

- Run the deployment 
	```	
	ansible-playbook gitlab_cluster_01_deploy.yml -i environments/
	```

#### Keycloak

- Run the deployment 
	```	
	ansible-playbook keycloak_01_deploy.yml -i environments/
	```

### Manage

#### HAProxy management
- Change the HAProxy configuration and/or update certificates
	+ Add/update the certificates in the ```certs/``` directory and ensure the proxy host host_var entry ```haproxy_certs_dir``` points to the correct location. Note, for HAProxy it is necessary to [concatenate the certificate and key files](https://www.meshcloud.io/en/blog/pem-file-layout-for-haproxy/).
	+ Update the configuration file in the ```files/``` directory and ensure the proxy host host_var entry ```haproxy_config_file``` points to the correct config file.
	+ Run the config/cert update playbook
	```
	ansible-playbook haproxy_02_config_cert_reload.yml -i environments/ 
	```
		 
+ Upgrade the HAProxy hosts
	+ Run the haproxy host upgrade playbook.
	```
	ansible-playbook haproxy_03_upgrade_host.yml -i environments/ 
	```

#### Gitlab management
- Change the Gitlab configuration (i.e. a gitlab.rb update)
	+ Update the group_vars for the gitlab cluster, and/or update the appropriate gitlab.rb template in the ```templates/``` directory.
	+ Run the config and secrets update playbook
	```
	ansible-playbook gitlab_cluster_02_update_config_secrets.yml -i environments/ 
	```
	+ Note, this playbook will also update the ```gitlab-secrets.json``` if this file changes. In normal usage this file is handled by the deployment and upgrade playbooks without administrator input required. The exception to this is if the gitlab is being restored from a previous instance. In this case, run the deployment playbook as normal, then place the ```gitlab-secrets.json``` from the instance being restored in the ```files/``` directory, specify the filename in the ```gitlab_secrets``` gitlab cluster group_vars variable, then run this playbook. 


- Upgrade Gitlab version
	+ Ensure the desired version of Gitlab is entered into the ```gitlab_version``` gitlab cluster group_vars variable. Or leave this variable blank to let the playbook determine the most recent Gitlab version release.
	+ Run the gitlab upgrade playbook
	```
	ansible-playbook gitlab_cluster_03_00_master_upgrade_gitlab.yml -i environments/ 
	```
	+ Note, the above playbook imports from 3 additional playbooks. These are 1) upgrading the gitlab application on the cluster hosts, 2) applying code patches to facilitate large file uploads to Gitlab, and 3) ensuring any upgrade related changes to ```gitlab-secrets.json``` are synced between hosts.  These can be run independently if required. 
	
- Upgrade the Gitlab cluster hosts
	+ Run the upgrade cluster host playbook
	```
	ansible-playbook gitlab_cluster_04_upgrade_host.yml -i environments/ 
	```

- Start a Gitlab backup
	+ Review the exclusions in the ```backup_skip``` gitlab cluster group_vars variable.
	+ Run the backup playbook
	```
	ansible-playbook gitlab_cluster_05_00_invoke_backup.yml -i environments/ 
	```
	+ Note, this invokes the ```gitlab-backup``` application. If you are expecting to be hosting large data volumes, it is recommended to exclude this data from ```gitlab-backup```, such as the files in the ```LFS``` and ```uploads``` stores. If any further backup steps are desired it is recommended to include these as additional playbooks imported by the backup playbook above. For example, in the Tübingen DataHUB, ```LFS``` and ```uploads``` are exluded from ```gitlab-backup```, and the S3 object store is more efficiently replicated using an additional playbook imported by  ```gitlab_cluster_05_00_invoke_backup.yml```. Keeping the backup processes together makes it easier to set up [automatic backups](#automate-it). 


#### Keycloak management
- Upgrade Keycloak version
	+ Ensure the desired version of Keycloak is entered into the ```keycloak_version``` host_vars variable. Or leave this variable blank to let the playbook determine the most recent Keycloak version release.
	+ Run the Keycloak backup and upgrade playbook
	```
	ansible-playbook keycloak_02_backup_upgrade.yml -i environments/ 
	```
	
- Upgrade Keycloak host
	+ Run the Keycloak host upgrade playbook
	```
	ansible-playbook keycloak_03_upgrade_host.yml -i environments/ 
	```

### Automate it
Included in the library is a playbook for managing systemd timers to run playbooks on a schedule. This can be used to run a playbook once, or to set up a recurring event. An full example set of timers are included as comments to set up a daily backup and monthly upgrades to the applications and hosts. To modify the timers, see this resource on [formatting systemd time strings](https://documentation.suse.com/smart/systems-management/html/systemd-working-with-timers/index.html#systemd-timer-types). Before being applied, timer strings can be [tested by systemd-analyze](https://documentation.suse.com/smart/systems-management/html/systemd-working-with-timers/index.html#systemd-timer-test).

- Run the systemd timer management playbook
	```
	ansible-playbook keycloak_03_upgrade_host.yml -i environments/
	``` 
+ The current timers can be viewed with the ```list-timers``` command:
	```
	systemctl list-timers
	```
+ Logs of the playbooks run via timers can be accessed via the systemd logging system.
	```
	journalctl -u [name of service]
	```
	+ For example, a log of scheduled backups can be viewed via:
	```
	journalctl -u ansible-gitlab-daily-backup
	```


### Requirements

#### Basic requirements

The DataHUB cluster requires the following resources:
- 1 host for the ansible controller
- 1 host instance for the reverse proxy
- 3+ hosts for the gitlab cluster hosts
- 1 internal network
- A public IPv4 and/or IPv6 address(es).
- 1 compute instance for keycloak (optional)


#### Gitlab cluster hosts
These minimum requirements are sourced from GitLab for version 18.3 (see https://docs.gitlab.com/ee/install/requirements.html, accessed 28.07.24)

 -  **CPU**. 8 vCPU is the recommended minimum, estimated to support up to 20 requests/s. 

 -  **Memory**. 16GB is the recommended minimum memory, estimated to support up to 20 requests/s.

- **Storage**. Dependant on the expected usage. The gitlab package itself on Linux requires 2.5GB. In the DataPLANT DataHUB we expect users to be uploading large files, so we provision storage volumes (3TB) for the gitlab host VMs, as well as a large capacity object store for LFS and other file uploads. 

#### Other hosts
The Ansible controller, reverse proxy and keycloak instances require minimal computing resources. A minimum of 2vCPUs, 4GB RAM and 20GB sotrage is recommended for each of these instances.

### Ansible
The heart of the project is the Ansible library. Ansible is a scripting language tailored to the deployment and management of computing resources. It is intended to be run from a central control node, which then deploys python scripts to carry out operations on remote nodes. Ansible is a **declarative** language. The administrator declares the desired state of the systems being managed, and the Ansible runtime ensures the systems being managed match this declaration. Actions are therefore only performed _if_ it is required to bring the system to the declared state.

As the administrator, the most important thing to know is that the operations are carried out using _playbooks_, which are the ```.yml``` files at the project top level alongside this Readme. Playbooks are executed by invoking the ```ansible-playbook``` python binary, specifying the playbook to run, and specifying an _inventory_, a  description of the computing resources to apply the playbook to. For example:
> ```ansible-playbook update-my-hosts-playbook.yml -i inventory-of-hosts.yml``` 

More about Ansible can be found in the:
- [Introduction to Ansible](https://docs.ansible.com/ansible/latest/getting_started/introduction.html)
- [Installation guide](https://docs.ansible.com/ansible/latest/installation_guide/index.html)
- [Guide to Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/index.html)
- [Guide to Ansible inventories](https://docs.ansible.com/ansible/latest/inventory_guide/index.html)


More detail and explanation is given in the deployment and management guides.


### Troubleshooting

#### Error running the Ansible playbook or vault
> ```
> Command 'ansible-playbook' not found, but can be installed with:
> sudo apt install ansible-core
>```
- Cause: The Ansible binaries are not in the user PATH.
- Solution:
	* Add the Ansible binaries to the user session PATH 
	```
	source /srv/ansible/dataplant-datahub-env/bin/activate
	```
	* **OR**, provide the full path to the Ansible binaries:
	 ```
	 /srv/ansible/dataplant-datahub-env/bin/ansible-playbook update-hosts-playbook.yml -i hosts-to-update.yml
	 ```
----