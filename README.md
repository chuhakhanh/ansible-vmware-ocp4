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

    dig master01.ocp4.example.com
        
    cp install-config.yaml /root/ocp4upi

    cd /root/ocp4upi
    openshift-install create manifests --dir=.
    openshift-install create ignition-configs --dir=.
    cp *.ign /var/www/html/openshift4/4.6.4/ignitions/
    chmod +r /var/www/html/openshift4/4.6.4/ignitions/*.ign