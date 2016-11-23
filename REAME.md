ansible-murdock
===============

Playbook to help Murdock deploy OCP 3.3 in his environment.

Overview
--------
The steps in this deployment process are:

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
  * Requires at least 100GB available from /root for untar-ing the Docker images
4. OpenShift master
5. OpenShift node1
6. OpenShift node2

Note that servers 3-6 will require an additional 100GB volume, the path for which is referenced by ```group_vars/all.yml``` variable, ```docker_volume```.

These hosts are put into the static inventory file in ```inventory/static/hosts``` in this fashion.

```
[openshift]
master.murdock.rhtps.io
node1.murdock.rhtps.io
node2.murdock.rhtps.io

[openshift_masters]
master.murdock.rhtps.io

[openshift_nodes]
node1.murdock.rhtps.io
node2.murdock.rhtps.io

[gitlab]
gitlab.murdock.rhtps.io

[registry]
registry.murdock.rhtps.io
```

Running the playbooks
---------------------

When you have yum populated with the required repos, use this process to deploy.

Notes:
* my_key.pem matches whatever is in your remote systems' nonroot user's ```.ssh/authorized_keys``` file.
* ```inventory/static/hosts``` has your six systems' hostnames grouped by role as above.

```
# ssh-agent bash
# ssh-add my_key.pem
# ansible-playbook -i inventory/static/hosts hostprep.yml 
```

After a successful run, set your GitLab's root password by navigating to its hostname in your browser.

Next, create an administrative user and set its password. See the [GitLab Documentation](https://docs.gitlab.com/ce/workflow/add-user/add-user.html) for details.

Now, you can complete the deployment.

```
# ansible-playbook -i inventory/static/hosts deploy.yml
```

When finished, you need to create static users on the OpenShift Master.

```
# ssh nonroot@master.murdock.rhtps.io
$ sudo su -
# htpasswd -b /etc/origin/master/htpasswd murdock mypassword
# oadm policy add-cluster-role-to-user cluster-admin murdock
```

Now, using this username and password, log into OpenShift and create some apps!

URL: ```https://master.murdock.rhtps.io:8443/console```