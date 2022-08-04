# ansible-vmware-okd-centos8

## Setup

### deploy cluster  

On deploy-1
    
    ansible-galaxy collection install community.crypto
    git clone https://github.com/chuhakhanh/ansible-vmware-okd-centos8
    cd /root/ansible-vmware-okd-centos8
    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=create"
    echo "10.1.17.253 utility" >> /etc/hosts
    ssh-copy-id root@utility
    
    ansible-playbook -i config/inventory prepare_node_all.yml
    ansible-playbook -i config/inventory prepare_node_utility.yml
### setup VM utility 
VM Boot Option 
- Firmware: BIOS

   
    dig master01.ocp4.example.com
 

    nmcli connection modify ens224 connection.zone internal
    nmcli connection modify ens192 connection.zone external
    firewall-cmd --zone=external --add-masquerade --permanent
    firewall-cmd --zone=internal --add-masquerade --permanent
    firewall-cmd --list-all --zone=internal
    firewall-cmd --list-all --zone=external
    
    firewall-cmd --add-port=53/udp --zone=internal --permanent
    firewall-cmd --add-port=53/tcp --zone=internal --permanent
    firewall-cmd --add-service=dhcp --zone=internal --permanent
    firewall-cmd --add-port=8080/tcp --zone=internal --permanent
    
    firewall-cmd --add-port=6443/tcp --zone=internal --permanent # kube-api-server on control plane nodes
    firewall-cmd --add-port=22623/tcp --zone=internal --permanent # machine-config server
    firewall-cmd --add-service=http --zone=internal --permanent # web services hosted on worker nodes
     firewall-cmd --add-service=https --zone=internal --permanent # web services hosted on worker nodes
 
    firewall-cmd --add-port=6443/tcp --zone=external --permanent # kube-api-server on control plane nodes
    firewall-cmd --add-service=http --zone=external --permanent # web services hosted on worker nodes
    firewall-cmd --add-service=https --zone=external --permanent # web services hosted on worker nodes
    firewall-cmd --add-port=9000/tcp --zone=external --permanent # HAProxy Stats

    firewall-cmd --zone=internal --add-service mountd --permanent
    firewall-cmd --zone=internal --add-service rpc-bind --permanent
    firewall-cmd --zone=internal --add-service nfs --permanent
    firewall-cmd --add-port=32700/tcp --zone=external --permanent # haproxy stats
    firewall-cmd --reload
    
    ssh -i /root/.ssh/id_rsa core@bootstrap
    watch 'ps -ef| grep -v "\["'

It's an ignition file problem.When we create a ignition file, we have to finish the installation with in 24 hours.Because the ignition files contains certificate and it will expires in 24 hours.