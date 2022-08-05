# ansible-vmware-okd-centos8


## Introduction

### References
    https://docs.okd.io/4.9/support/troubleshooting/troubleshooting-installations.html
    https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-bare-metal-network-customizations.html#installation-user-infra-machines-iso_installing-bare-metal-network-customizations

### Software version

| Software | Version |
| --- | --- |
| `openshift` | *4.6.1* |
| `rhcos` | *4.6.1* |

### Architect

Process of installation 

![Deploy process](./media/pics/process1.png)

Diagram of system

![Diagram](./media/pics/diagram1.png)
## Setup


### deploy cluster  

On deploy-1
    
    ansible-galaxy collection install community.crypto
    git clone https://github.com/chuhakhanh/ansible-vmware-okd-centos8
    cd /root/ansible-vmware-okd-centos8
    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=create"
    echo "10.1.17.253 utility" >> /etc/hosts
    ssh-copy-id root@utility

Prepare environment such as local repository, hosts file    
    
    ansible-playbook -i config/inventory prepare_node_all.yml

### Setup the utility node    

Setup required software node utility such as: dns, dhcp ...
    
    ansible-playbook -i config/inventory prepare_node_utility.yml
    
Prepare ignition for setup OCP cluster

First edit pull secret variables file vars/vmw_env.yml. Edit my_pull_secret: '{"auths":....}'

Run playbook

    ansible-playbook -i config/inventory prepare_ocp_ignition.yml
### Setup VM in OCP cluster using Method ISO Installation

References: 
    
    https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-bare-metal-network-customizations.html#installation-user-infra-machines-iso_installing-bare-metal-network-customizations

Setup VM Installing a cluster on bare metal with network customizations - Installing on bare metal | Installing | OpenShift Container Platform 4.6

Download file and put to iso datastore 

    https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.6/4.6.1/rhcos-installer.x86_64.iso 

Settings VM and install

    - VM Boot Option>Firmware: BIOS
    - CDROM> Select rhcos-installer.x86_64.iso

Set timezone for VM

    sudo timedatectl set-timezone Asia/Saigon

For bootstrap VM
    
    sudo coreos-installer install /dev/sda --insecure-ignition --ignition-url=http://192.168.50.254:8080/openshift4/4.6.4/ignitions/bootstrap.ign 
    
For master VM
    
    sudo coreos-installer install /dev/sda --insecure-ignition --ignition-url=http://192.168.50.254:8080/openshift4/4.6.4/ignitions/master.ign 

For worker VM

    sudo coreos-installer install /dev/sda --insecure-ignition --ignition-url=http://192.168.50.254:8080/openshift4/4.6.4/ignitions/worker.ign 

Reboot 


    ssh -i /root/.ssh/id_rsa core@bootstrap
    watch 'ps -ef| grep -v "\["'

Note: 

    - It's an ignition file problem.When we create a ignition file, we have to finish the installation with in 24 hours.Because the ignition files contains certificate and it will expires in 24 hours.
    - If you recreate OCP and recreate ignition file, remove hidden files: /root/ocp4ui/.openshift_install_state.json 

    ssh core@<bootstrap_fqdn> journalctl -b -f -u bootkube.service
    ssh core@<bootstrap_fqdn> 'for pod in $(sudo podman ps -a -q); do sudo podman logs $pod; done'
     
    export KUBECONFIG=/root/ocp4upi/auth/kubeconfig
    openshift-install --dir=ocp4upi wait-for install-complete --log-level=debug

    https://docs.openshift.com/container-platform/4.6/post_installation_configuration/node-tasks.html
    oc get csr
    oc adm certificate approve csr-7lnxb
    oc get nodes
    oc get co

### Login

After install 

    http://10.1.17.253:32700/

Edit hosts file

    10.1.17.253 console-openshift-console.apps.ocp4.example.com oauth-openshift.apps.ocp4.example.com

Login to console 

    https://console-openshift-console.apps.ocp4.example.com/monitoring/dashboards/grafana-dashboard-etcd