# SolidFire Kubernetes Baremetal #

Deploy Kubernetes clusters with Kubespray on bare metal (physical servers) including virtual machines.

### Description ###

Automated install Kubernetes clusters using Kubespray.  The clusters are designed to be built on virtual machines or bare metal.

The cluster will use the following components:  
Control plane container engine: *docker*  
Container network interface: *calico*  
Storage driver: *overlay2*  

Estimated time to complete: 1 hr

Kubespray git repo:  `https://github.com/kubernetes-incubator/kubespray`

### Requirements ###

General requirements:

* Control node: Where the Kubespray commands are run (i.e. laptop or jump host).
* Cluster machines: Minimum of one, but at least three are recommended
* Operating system: Ubuntu 16.04   (CentOS 7 support upcoming under consideration)

### Prepare Control Node ###

Prepare control node where management tools are installed.  A laptop computer will be sufficient.

MacOS or Linux:

1. Install required packages.  Ansible v2.4 (or newer) and python-netaddr is installed on the machine that will run Ansible commands.

    `$ pip2 install ansible kubespray`  

2. Clone repo with ansibles

    `$ cd; git clone https://bitbucket.org/solidfire/kubespray-and-pray`

3.  To-do  
    Known hosts??  Make connection first?
    Might need to log in and make a ssh connection which will create .ssh dir.

### Install Components ###

Perform the following steps on the control node where ansible command will be run from.  Define the nodes, etcds and masters as appropriate.

1. Hostname resolution.

    Ensure that the names are resolvable in DNS or are listed in local hosts file.

    The control node and all cluster vm's must have DNS resolution or /etc/hosts entries.  IP addresses may be used if you must.

2. Run command to generate inventory file (*~/.kubespray/inventory/inventory.cfg*) which defines the target nodes.  If there are too many hosts for command-line, run the kubespray prepare command with a minimal set of hosts then add to the resulting inventory.cfg file.

    `$ cd ~/kubespray-and-pray`
    `$ kubespray prepare --nodes k8s0 k8s1 k8s2 --etcds k8s0 k8s1 k8s2 --masters k8s0`

    The file ansible.cfg defines the inventory file as *~/.kubespray/inventory/inventory.cfg*.  This will be the default inventory file when ansible is run. 
    If multiple network adapters are present, then define the one to use by adding lines defining ansible_ssh_host to top of file.
    '''
    k8s0 ansible_ssh_host=10.117.31.20
    k8s1 ansible_ssh_host=10.117.31.21
    k8s2 ansible_ssh_host=10.117.31.22
    '''

3. Bootstrap ansible by installing Python.  Note that ansible.cfg defines the inventory file as *~/.kubespray/inventory/inventory.cfg*.  This will be the default inventory file when ansible is run.  Supply SSH password. 

    `$ ansible-playbook bootstrap-ansible.yml -k -K`

4. Allow user solidfire to sudo without password.

    `$ ansible-playbook solidfire-sudo.yml -k -K`

5. Run pre-install step.

    `$ ansible-playbook kubespray-pre.yml`

6. Create container storage volume.

    `$ ansible-playbook create-volume.yml`

7. Edit cluster parameters.

    `$ cp -a ~/.kubespray/inventory/sample/group_vars ~/.kubespray/inventory`

    `$ vi ~/.kubespray/inventory/group_vars/all.yml`

    Uncomment the following line:
    `docker_storage_options: -s overlay2`  

    `$ vi ~/.kubespray/inventory/group_vars/k8s-cluster.yml`

    Modified helm_enabled:
    `helm_enabled: true`
 
8. Deploy Kubespray.  Ansible is run on all nodes to install and configure Kubernetes cluster.
 
    `$ kubespray deploy -u solidfire`
    
Congratulations!  You're cluster is running.  On a master node, run `kubectl get nodes` to validate.

### Docker Volume ###

From the control node, run post-install steps.  This includes configuring Docker logical volume.  Raw storage volume (defaults to /dev/sdb) will be used for container storage.

1. Run ansible post-install tasks.

    `$ ansible-playbook kubespray-post.yml`

### Gluster Filesystem ###

Configure hyper-converged storage solution consisting of a Gluster distributed filesystem running as pods in the Kubernetes cluster.  Gluster cluster is managed by Heketi.  Raw storage volume (defaults to /dev/sdc) will be used for GlusterFS.

Heketi install procedure: `https://github.com/heketi/heketi/blob/master/docs/admin/install-kubernetes.md`

1. Run ansible to install kernel modules and glusterfs client.

    `$ ansible-playbook heketi-gluster/gluster-pre.yml`

2. Create GlusterFS daemonset.

3. Heteki ...

### Kubernetes Permissions ###

***WARNING... Insecure permissions for development only!***

1. Log onto a Kubernetes master node.

2. Run the following on the master node.  

    `$ kubectl -n kube-system edit service kubernetes-dashboard`

3. Identify the line:  
    `type: CluserIP`  
    Change to:  
    `type: NodePort`  

4. Permissive admin role.  
    Kubernetes RBAC: `https://kubernetes.io/docs/admin/authorization/rbac/`

    WARNING: The following policy allows ALL service accounts to act as cluster administrators. Any application running in a container receives service account credentials automatically, and could perform any action against the API, including viewing secrets and modifying permissions. This is not a recommended policy... On the other hand it works like charm for dev!

    `$ kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=admin --user=kubelet --group=system:serviceaccounts`

5. Get Kubernetes master IP address.  
    `kubectl cluster-info`

6. Get dashboard port.  
    `kubectl -n kube-system get service kubernetes-dashboard`

7. Access dashboard with url.  
    `https://<master_ip>:<dashboard_port>/`

### Contact ###

* NetApp SolidFire Central Engineering
* Maintainer:  ronald.compos@netapp.com
