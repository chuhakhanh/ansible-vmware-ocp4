# ansible-vmware-okd-centos8


## Introduction

This guide to install a OCP4 with tested version 4.6.1. The lab environment is created on VMware vsphere environment. 
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
  
On a vsphere environment, create a several VM  cluters with different name but same IP address and MAC address. So we do not need to reconfigure the utility VM include DNS, haproxy. However, we can use only 1 single VM cluster at one time.
  
    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=create" -e "rhcos_ver=4.10.16"
    echo "10.1.17.253 utility" >> /etc/hosts
    ssh-copy-id root@utility

Or manage the cluster
    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=destroy" -e "rhcos_ver=4.10.16"

Prepare environment such as local repository, hosts file    
    
    ansible-playbook -i config/inventory prepare_node_all.yml
    

### Setup the utility node    

Setup required software node utility such as: dns, dhcp ...
    
    ansible-playbook -i config/inventory prepare_node_utility_service.yml 

There are severals openshift version can be deploy on node utility

    ansible-playbook -i config/inventory prepare_node_utility_ocp.yml -e "rhcos_ver=4.10.16"
    
In node utility, prepare ignition for setup OCP cluster

    mkdir /root/4.10.26; cd /root/4.10.26
    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.10.26/openshift-client-linux.tar.gz 
    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.10.26/openshift-install-linux.tar.gz
    ln -s /root/4.10.26/kubectl /usr/local/bin/kubectl
    ln -s /root/4.10.26/oc /usr/local/bin/oc
    ln -s /root/4.10.26/openshift-install /usr/local/bin//openshift-install

Get pull secret variables from at https://console.redhat.com/openshift/install/metal/installer-provisioned/, then copy into my_pull_secret: '{"auths":....}' in prepare_ocp_ignition.yml file

    ansible-playbook -i config/inventory prepare_ocp_ignition.yml -e "rhcos_ver=4.10.16"
    
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

OCP Version: 4.6.1
For bootstrap VM
    
    sudo coreos-installer install /dev/sda --insecure-ignition --ignition-url http://192.168.50.254:8080/openshift4/4.6.4/ignitions/bootstrap.ign 
    
For master VM
    
    sudo coreos-installer install /dev/sda --insecure-ignition --ignition-url http://192.168.50.254:8080/openshift4/4.6.4/ignitions/master.ign 

For worker VM

    sudo coreos-installer install /dev/sda --insecure-ignition --ignition-url http://192.168.50.254:8080/openshift4/4.6.4/ignitions/worker.ign 

Reboot 

    
    rhcos_ver=4.10.16
    rm -f /root/.ssh/known_hosts
    ssh -i /root/.ssh/"$rhcos_ver"/id_rsa core@bootstrap
    ssh -i /root/.ssh/"$rhcos_ver"/id_rsa core@master01
    watch 'ps -ef| grep -v "\["'
    
    export KUBECONFIG=/root/ocp4upi/"$rhcos_ver"/auth/kubeconfig


Note: 

- It's an ignition file problem.When we create a ignition file, we have to finish the installation with in 24 hours.Because the ignition files contains certificate and it will expires in 24 hours.
- If you recreate OCP and recreate ignition file, remove hidden files: /root/ocp4ui/.openshift_install_state.json 

Troubleshooting while bootstraping:

    ssh core@bootstrap 
    journalctl -b -f -u release-image.service -u bootkube.service
    journalctl -b -f -u bootkube.service
    for pod in $(sudo podman ps -a -q); do sudo podman logs $pod; done

References: 

    https://docs.openshift.com/container-platform/4.6/post_installation_configuration/node-tasks.html

For clusters running on platforms that are not machine API enabled, such as bare metal and other user-provisioned infrastructure, you must implement a method of automatically approving the kubelet serving certificate requests (CSRs). [Review logss in logs/utility.log to review output from node utlity about bootstrap process](logs/utility.log)
Example:
After there are some error logs, and request pending. After approve all the request, all logs is cleared.
E0808 15:40:48.752187   58531 reflector.go:307] k8s.io/client-go/tools/watch/informerwatcher.go:146: Failed to watch *v1.ClusterVersion: the server is currently unable to handle the request (get clusterversions.config.openshift.io)
DEBUG Still waiting for the cluster to initialize: Cluster operator authentication is reporting a failure: WellKnownReadyControllerDegraded: need at least 3 kube-apiservers, got 2 

On node utility

    rhcos_ver=4.10.16
    export KUBECONFIG=/root/ocp4upi/"$rhcos_ver"/auth/kubeconfig
    openshift-install --dir=/root/ocp4upi/"$rhcos_ver" wait-for install-complete --log-level=debug
 
    oc get csr
    oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
    oc get nodes
    oc get co

On haproxy

    http://10.1.17.253:32700/
### After install, login to OCP4

Edit hosts file

    10.1.17.253 console-openshift-console.apps.ocp4.example.com oauth-openshift.apps.ocp4.example.com

Login to console 

    https://console-openshift-console.apps.ocp4.example.com/monitoring/dashboards/grafana-dashboard-etcd
    user: kubeadmin
    password: cat /root/ocp4upi/"$rhcos_ver"/auth/kubeadmin-password