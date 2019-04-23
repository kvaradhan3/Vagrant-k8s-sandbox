# Creating VMs for playing with kubernetes

Borrowed liberally from
[<http://www.opencontrail.org/installing-kubernetes-opencontrail/>]
but the blog is very out of date and newerinstructions are elsewhere.
We use this blog to create our VMs to the topologhy below:

```

      +-----------+                    +-----------+
      |k8s-node-01|<------------------>|k8s-node-02|
      +----\------+                    +------/----+
            \                                /
             \                              /
              \                            /
               \                          /
                \                        /
                 \                      /
                  \                    /
                   \  +------------+  /
                    \ |            | /
                     -| k8s-master |-
                      |            |
                      +-----+------+
                            |
                            |
                            |
                            |
                            |
                            |
                            |
                      +-----+-------+
                      | k8s-gateway |
                      +-------------+

```
Assuming we have vagrant and a centos-7 image with a reasonable
storage space, we create 4 VMs in the following topology:

`servers.yaml` drives both vagrant and potentially, kubernetes
provisioning through ansible.  Specify the default networking parameters,
and IP addresses for each of the hosts here.  You ought to be able to
leave the rest as they were.
```
---
defaults:
  network:
    :netmask:    255.255.252.0
    :bridge:     eno1
    :type:       public_network
    :default-gw: 10.84.14.254

servers:
- name: k8s-master
  network:
    - :ip:         10.84.14.114

- name: k8s-gateway
  network:
    - :ip:         10.84.14.111

- name: k8s-node-01
  network:
    - :ip:         10.84.14.115

- name: k8s-node-02
  network:
    - :ip:         10.84.14.116
```
The postconfig.yaml file describes ansible rules for the initial
provisioning of the servers and does the following tasks:
```
    - name: read servers.yaml into servers list.
    - name: Set hostname
    - name: Permit Root Login
    - name: Restart sshd
    - name: overlay /etc/resolv.conf
    - name: Populate /etc/hosts from servers.yaml
    - name: install a few basic tools
      yum:
        name:
          - net-tools
          - traceroute
          - git
          - lsof
          - yum-utils
          - epel-release
          - ruby
    - name: set the root password, if it is given
    - name: penultimate check, is postconfig-custom.yaml present
    - name: ultimately, if present, execute it
```
If we need additional post-provisioning, add to a postconfig-custom.yaml file.


With these, we create our VMs as:
```
vagrant --root-password 'XXXXXXX' up
```
We also use ansible to setup a hosts file that contains the names of
all the vagrant hosts using this ansible script.  We dont really need
this script, since the config.yaml file will populate the hosts file
from the initial servers.yaml list.  There is one difference though.
The servers.yaml file only adds the first interface we have added into the
hosts file `(servers[*].network[0].:ip)`.  If we either added multiple,
or just took the vagrant defaults, or added the vhost0 interface that
contrail w/openstack orchestration uses, we may need to use this one
as well.  The two scripts are idempotent and have no side effects,
so are safe to use together.
```
% ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory mk-Hosts-fil.yaml
```

So we now have 4 identical VMs, and we can try and setup kunernetes
in one or more of them.  There are two ways to setup kubernetes and contrail.

Setting up kubernetes in the traditional way, with any CNI now is:

```
% ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory -e cni=contrail kubernetes.yaml
```
The default CNI is flannel.  You can also start kubernetes w/no CNI by specifying `-e cni=`.
The contrail CNI in this model would be installed as a daemon-set, so it naturally fits the k8s model of deployment.

If you are using contrail as a CNI, you need to provide some additional
parameters on where contrail can fetch its containers.  These are
specified in `"{{ playbook_dir }}/roles/master/vars/main.yml"`.
These variables are:
```
---
# vars file for k8s-common
hub_userid:         "FILL THESE"
hub_password:       "FILL THESE"
hub_email:	    "FILL THESE"

CONTRAIL_REPO:      'hub.juniper.net/contrail'
CONTRAIL_RELEASE:   '5.0.2-0.360'
```

This model of deploying contrail as a CNI however does not work as well as could be expected.

An alternate way of doing this is the traditional contrail-ansible-deplpyer model of contrail.
Here, we skip the ansible based kubernetes setup, and instead use the contrail+k8s playbook as

1. rerun vagrant up with provisioners.
	`vagrant --root-password=<password> up --provision`
2. Setup contrail and k8s as:
        `ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory playbooks/contrail+k8s.yaml`
	
At a later point, to update the inventory,
1. Connect to the master, 
	`vagrant ssh k8s-master`
2a. on the master, become root
	`sudo -s`
2b. cd into the prepared deployer directory
	`cd ~/deployer`
2c. update the instances.yaml file in that directory, and then
2d. Run the deployer script there,
	`./deploy-contrail.sh`
