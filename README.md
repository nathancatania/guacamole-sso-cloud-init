# Automated Guacamole (with SAMl & SSL) Deployment using Cloud-Init
**Version: 0.1.0**

For full instructions, [please see my blog post](https://nathancatania.com/). This has been tested with both Azure AD and Okta.

## What is this?
This is a User-Data configuration file for Cloud-Init that will automatically deploy and configure an instance of Apache Guacamole (fully integrated with SSL and SAML for authentication) in AWS, Azure or GCP; eliminating the need for any manual configuration from the command-line.

This means that all you need to do is deploy a standard Ubuntu VM in any of these providers, and as long as this configuration is pasted into the VM config during creation, within 10 minutes, user's will be able to SSO to a fresh and secure instance of Guacamole.

## What is Cloud-Init?
[Cloud-Init](https://cloudinit.readthedocs.io/en/latest/index.html) is a piece of software included out of the box in most common Linux Distros (eg: Ubuntu, RHEL), and is used to automatically provision initialize required configuration on first boot/deployment of a VM.

On instance boot, cloud-init will identify the cloud it is running on, read any provided metadata from the cloud, and initialize the system accordingly. This may involve setting up the network and storage devices, configuring SSH access keys, and setting up many other aspects of a system. Later, cloud-init will parse and process any optional user or vendor data that was passed to the instance.

## Why is this useful?
Guacamole is a useful tool (especially when coupled with a ZTNA solution) that can facilitate access to servers, all through the user's browser. Unfortunately, Guacamole can be a little tricky to get up and running, ESPECIALLY with both SAML authentication and SSL. This script will allow you to have a fresh Guacamole instance, fully integrated with SAML, in minutes - just by changing a few parameters at the top of the config.
I've personally done all of the debugging and config trial and error for you (and to get Gucamole reliably working with SAML, there was a lot of it!)

## What does the script/configuration do?
Essentially it automates the manual configuration process I walk through in a blog post, here. At a high level, it:

1. Updates the system and all packages
2. Creates a directory structure to hold all required config files
3. Installs Docker and Docker Compose
4. Creates all required config files
5. Downloads the Guacamole SAML extension
6. Brings up all Guacamole services correctly configured
7. Automatically creates self-signed SSL certificates for the specified domain that the instance is accessed from.

When added to a new VM instance, cloud-init can take time to fully run (~10 minutes) depending on connection speed, so even if you can SSH to the VM straight away, cloud-init is still likely applying the configuration. Be patient if it looks like it hasn't worked in the first few minutes of a VMs life.

# How do I use this?
1. [Access the configuration file here](https://github.com/nathancatania/guacamole-sso-cloud-init/blob/main/user-data.yaml) and copy the text.
2. Edit and change the appropriate fields in the `user-data`.yml file. As a minimum, you MUST specify where indicated:
  * The FQDN/domain that Guacamole will be accessed at, eg: `guac.company.com`
  * The metadata URL for your IdP (Azure AD, Okta, etc) - This contains the settings needed by Guacamole to redirect user's to SSO.
  * You will also want to specify an SSH key to that you can connect to the VM.
3. Create an Ubuntu VM within your IaaS environment. This was validated on Ubuntu Server 20.04 LTS.
4. During the VM creation workflow, paste in the configuration text into a field marked as cloud-init/custom-data/user-data. This is different for every platform (see specific instructions below).
5. Wait a few minutes. After some time, SSH to the VM (unless changed in the config, the default user is `guacamole`). You should now see the Gucamole containers running. Hint: Run `docker ps` to check.
6. Navigate to the Guacamole UI, `https://<your-specified-fqdn>/guacamole, and you should be redirected to SSO.

Congratulations, you now have a working Guacamole instance, secured with SSL and integrated with SAML!

## AWS / EC2
* For AWS EC2, when creating a VM instance, scroll down and under **Advanced Details**, simply paste the configuration into the **User-Data**.
* Don't worry about explicitly setting an SSH identity/key when configuring the VM details - the configuration data can do this for you under the default user (`guacamole` by default unless changed).
* Don't forget to add your FQDN and Azure AD/Okta Metadata URL.

![aws](https://i.imgur.com/vc7POtl.png)

## Azure
* For Azure, when creating a VM instance, select the **Advanced** tab, and paste the configuration into the **Custom data** field under the **Custom data and cloud init** sub-heading.
* Don't worry about explicitly setting an SSH identity/key when configuring the VM details - the configuration data can do this for you under the default user (`guacamole` by default unless changed).
* Don't forget to add your FQDN and Azure AD/Okta Metadata URL.

![azure](https://i.imgur.com/2GetPUq.png)

## GCP
* For Google Cloud, when creating a VM instance, open the **Management** tab, and add a new item under **Metadata**.
* Set the **key** to be `user-data` exactly! For the value, paste in the configuration.
* Don't forget to add your FQDN and Azure AD/Okta Metadata URL.

![gcp](https://i.imgur.com/6YLrF9f.png)

# Who are you?
I'm a Netskope Solutions Engineer that created this to make everyone's lives easier!

This script is NOT officially supported by Netskope, and by using it, **you agree to use it at your own risk**.

Any of my own thoughts posted here are my own and do not reflect those of Netskope.

# Validation / Troubleshooting Steps
If you're not sure whether this has worked properly, do the following:

## Connect to the VM
SSH to the VM using the `guacamole` user, eg: `ssh -i <ssh-key> guacamole@<vm-ipaddress>`

* Note: You'll only be able to SSH to the VM if you also specified an SSH key or identity in the configuration.
  
## Check the status of cloud-init
Check the status of the cloud-init service: `cloud-init status`

```
guacamole@i-097b1b0c65d4ef774:~$ cloud-init status
status: done
```

If the status shows `status: running`, cloud-init is still configuring the VM. Either wait a few more minutes or proceed below.

## Check the cloud-init logs

  * `/var/log/cloud-init-output.log` is the live output as the configuration is being applied. Very useful to observe this file if your cloud-init status is still `status: running`. You can also run `sudo tail -f /var/log/cloud-init-output.log` if cloud-init is still running to see the live output as this log is written to.
  * `/var/log/cloud-init.log` is the log of the overarching cloud-init process itself.
 
```
guacamole@i-097b1b0c65d4ef774:~$ sudo cat /var/log/cloud-init-output.log
[...snip...]
Reading package lists...
Building dependency tree...
Reading state information...
The following packages will be REMOVED:
  libfwupdplugin1
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 459 kB disk space will be freed.
(Reading database ... 92928 files and directories currently installed.)
Removing libfwupdplugin1:amd64 (1.5.11-0ubuntu1~20.04.2) ...
Processing triggers for libc-bin (2.31-0ubuntu9.7) ...
```

## Check the applied configuration
`/var/lib/cloud/instance/user-data.txt` is a copy of the configuration template you pasted in during VM creation. You can see what this looks like in a rendered/finished state (ie: the *exact* configuration that cloud-init applies) through the following command: `sudo cloud-init devel render /var/lib/cloud/instance/user-data.txt`

Unrendered template example:
```
guacamole@i-097b1b0c65d4ef774:~$ sudo cat /var/lib/cloud/instance/user-data.txt
 - 'docker run --rm guacamole/guacamole:{{ guac_tag }} /opt/guacamole/bin/initdb.sh --mysql > /home/{{ default_user }}/configs/guacdb/guacdb.sql'
 - 'chown -R {{ default_user }}:{{ default_user }} /home/{{ default_user }}/configs'
 - 'docker compose -f /home/{{ default_user }}/docker-compose.yml up -d'
```

Rendered template example:
```
guacamole@i-097b1b0c65d4ef774:~$ sudo cloud-init devel render /var/lib/cloud/instance/user-data.txt
[...snip...]
 - 'docker run --rm guacamole/guacamole:1.4.0 /opt/guacamole/bin/initdb.sh --mysql > /home/guacamole/configs/guacdb/guacdb.sql'
 - 'chown -R guacamole:guacamole /home/guacamole/configs'
 - 'docker compose -f /home/guacamole/docker-compose.yml up -d'
```

