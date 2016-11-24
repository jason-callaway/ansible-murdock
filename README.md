ansible-murdock
===============

Playbook to help Murdock deploy OCP 3.3 in his environment.

Overview
--------
The steps in this deployment process are:

* Provision your hosts and their extra volumes
* Set up your yum repos based on the tarballs you were provided
* Modify the variables in ```group_vars/all.yml``` to match your environment
* Run the ```hostprep.yml``` playbook
* Set the GitLab root password, create a user and set the user's password
* Run the ```deploy.yml``` playbook

Required Hosts
--------------

You'll need a total of 6 RHEL VMs, running at least 7.2.

1. Yum server (it's ok to reuse an existing yum server for this). Required repos:
  * rhel-7-server-rpms
  * rhel-7-server-extras-rpms
  * rhel-7-server-ose-3.3-rpms
2. GitLab server
3. Docker registry
  * Requires at least 100GB available from the ```tarball_untar_dir``` directory for untar-ing the Docker images
4. OpenShift master
5. OpenShift node1
6. OpenShift node2

Note that servers 3-6 will require an additional 100GB volume, the path for which is referenced by ```group_vars/all.yml``` variable, ```docker_volume```.

These hosts are put into the static inventory file in ```inventory/static/hosts``` in this fashion.

```
[master]
master.murdock.rhtps.io

[node1]
node1.murdock.rhtps.io

[node2]
node2.murdock.rhtps.io

[gitlab]
gitlab.murdock.rhtps.io

[registry]
registry.murdock.rhtps.io
```

Prep work
---------

Before you run the playbooks, there are a few prep steps to take:

1. Set up the ```hosts``` file described above
2. Set the variables in ```group_vars/all.yml```
  * The ```nonroot_username``` variable is the name you'll log into your servers with via ssh.
  * The ```yum_server``` variable is the FQDN or IP address of your yum server.
  * The ```docker_volume``` variable is the device path for the secondary disk that has been presented to your VMs. As an example, in AWS, this is typically ```/dev/xvdb```.
3. Copy the GitLab omnibus installer to the ```gitlab_rpm_path``` directory 
4. Copy the ```ose33_images.tar.gz``` file into the ```docker_images_path``` directory
5. Determine your apps subdomain. In the example we're using ```apps.murdock.rhtps.io```. This needs to exist as a ```*``` entry or "wildcard" in DNS that points to your master's FQDN. See the [OpenShift Docs](https://docs.openshift.com/container-platform/3.3/install_config/install/prerequisites.html#wildcard-dns-prereq) for details. 

Running the playbooks
---------------------

When you have yum populated with the required repos, use this process to deploy.

Notes:
* ```hostkey``` matches whatever is in your remote systems' nonroot user's ```.ssh/authorized_keys``` file
* ```inventory/static/hosts``` has your six systems' hostnames grouped by role as above

```
# ssh-agent bash
# ssh-add hostkey
# ansible-playbook -i inventory/static/hosts hostprep.yml 
```

If you need to re-run the playbook, time can be saved by:

```
# ansible-playbook -i inventory/static/hosts hostprep.yml --skip-tags=update_reboot
```

After a successful run, it's time to deploy OpenShift. SSH to the master and run:

```
# ansible-playbook -i /root/hosts /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml
```

When finished, you need to create static users on the OpenShift Master.

```
# ssh nonroot@master.murdock.rhtps.io
$ sudo su -
# htpasswd -b /etc/origin/master/htpasswd murdock mypassword
# oadm policy add-cluster-role-to-user cluster-admin murdock
```

Now, using this username and password, log into OpenShift.

URL: ```https://master.murdock.rhtps.io:8443/console```

GitLab needs to be configured in order to create apps. This is a manual process.

* Set your GitLab's root password by navigating to its hostname in your browser.
* Create an administrative user and set its password. See the [GitLab Documentation](https://docs.gitlab.com/ce/workflow/add-user/add-user.html) for details.
* Also add the SSH public key, ```mykey.pub``` located in this repo's directory, that was created by the previous playbook. The process for adding a key is in the [GitLab Documentation](https://docs.gitlab.com/ee/gitlab-basics/create-your-ssh-keys.html).
* Create projects for the OpenShift repos in ```ose33_repos.tar.gz``` and git push the repos into those projects. Example:
  * Create a public project called ```cakephp-ex```
  * Untar the ```ose33_repos.tar.gz``` tarball
  * ```cd cakephp-ex```
  * Now set the upstream with ```git remote remove origin; git remote add origin git@gitlab.murdock.rhtps.io:murdock/cakephp-ex.git```
  * And push the code into GitLab ```git push -u origin master```
* Now when creating apps in OpenShift you can point to your GitLab repos
  
