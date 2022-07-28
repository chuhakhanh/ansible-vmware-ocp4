# ansible-vmware-okd-centos8

## Introduction

### links and references
https://computingforgeeks.com/how-to-deploy-openshift-container-platform-on-kvm/


### description 
    bastion.ocp4.example.com                        10.1.17.100    192.168.100.254                 
    bootstrap.ocp4.example.com	52:54:00:a4:db:5f	10.1.17.90     192.168.100.10 
    
    master01.ocp4.example.com	52:54:00:8b:a1:17	10.1.17.91     192.168.100.11
    master02.ocp4.example.com	52:54:00:ea:8b:9d	10.1.17.92     192.168.100.12
    master03.ocp4.example.com	52:54:00:f8:87:c7	10.1.17.93     192.168.100.13
    
    worker01.ocp4.example.com	52:54:00:31:4a:39	10.1.17.94     192.168.100.14
    worker02.ocp4.example.com	52:54:00:6a:37:32	10.1.17.95     192.168.100.15
    worker03.ocp4.example.com	52:54:00:95:d4:ed	10.1.17.96     192.168.100.16

    Virtual Machine | Operating System |
     --- | --- 
    Bootstrap | RHCOS	

My Lab environment variables

    OpenShift 4 Cluster base domain: example.com ( to be substituted accordingly)
    OpenShift 4 Cluster name: ocp4 ( to be substituted accordingly)
    OpenShift KVM network bridge: openshift4
    OpenShift Network Block: 192.168.100.0/24 => 10.1.0.0/16
    OpenShift Network gateway address: 192.168.100.1 => 10.1.0.1
    Bastion / Helper node IP Address (Runs DHCP, Apache httpd, HAProxy, PXE, DNS) – 192.168.100.254
    NTP server used: time.google.com  

## Setup

### deploy basion VM 

On deploy-1
    
    git clone https://github.com/chuhakhanh/ansible-vmware-okd-centos8
    cd /root/ansible-vmware-okd-centos8
    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=create"
    
    cp -u config/hosts /etc/hosts
    chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory
    
    ansible-playbook -i config/inventory prepare_all_node.yml
### setup bastion VM

Step 3: Install Ansible and Configure variables on Bastion / Helper node
On bastion

    yum -y install git ansible vim wget curl bash-completion tree tar libselinux-python3
    cd /root
    git clone https://github.com/chuhakhanh/jmutai_ocp4_ansible.git
    cd /root/ocp4_ansible
    
 Step 4: Install and Configure DHCP serveron Bastion / Helper node

    yum -y install dhcp-server
    systemctl enable dhcpd
    mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bak
    ansible-playbook tasks/configure_dhcpd.yml

Step 4: Configure OCP Zone on Bind DNS Serveron Bastion / Helper node

    yum -y install bind bind-utils 
    systemctl enable named
    vim /usr/local/bin/set-dns-serial.sh
    #!/bin/bash
    dnsserialfile=/usr/local/src/dnsserial-DO_NOT_DELETE_BEFORE_ASKING_CHRISTIAN.txt
    zonefile=/var/named/zonefile.db
    if [ -f zonefile ] ; then
        echo $[ $(grep serial ${zonefile}  | tr -d "\t"" ""\n"  | cut -d';' -f 1) + 1 ] | tee ${dnsserialfile}
    else
        if [ ! -f ${dnsserialfile} ] || [ ! -s ${dnsserialfile} ]; then
            echo $(date +%Y%m%d00) | tee ${dnsserialfile}
        else
            echo $[ $(< ${dnsserialfile}) + 1 ] | tee ${dnsserialfile}
        fi
    fi
    ##
    ##-30-
    sudo chmod a+x /usr/local/bin/set-dns-serial.sh
    ansible-playbook tasks/configure_bind_dns.yml
    dig @127.0.0.1 -t srv _etcd-server-ssl._tcp.ocp4.example.com
    nmcli connection modify ens192 ipv4.dns "10.1.17.100"
    nmcli connection reload
    nmcli connection up ens192
    host bootstrap.ocp4.example.com

Step 5: Setup TFTP Serviceon Bastion / Helper node

    yum -y install tftp-server syslinux
    vim /etc/systemd/system/helper-tftp.service
[Unit]
Description=Starts TFTP on boot because of reasons
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/start-tftp.sh
TimeoutStartSec=0
Restart=always
RestartSec=30

[Install]
WantedBy=default.target

tee /usr/local/bin/start-tftp.sh<<EOF
#!/bin/bash
/usr/bin/systemctl start tftp > /dev/null 2>&1
##
##
EOF

    chmod a+x /usr/local/bin/start-tftp.sh
    systemctl daemon-reload
    systemctl enable --now tftp helper-tftp
    mkdir -p  /var/lib/tftpboot/pxelinux.cfg
    cp -rvf /usr/share/syslinux/* /var/lib/tftpboot

Obtain the RHEL kernel, initramfs, and rootfs files from the RHCOS image mirror page. The three main files to be downloaded:
kernel: rhcos-<version>-live-kernel-<architecture>
initramfs: rhcos-<version>-live-initramfs.<architecture>.img
rootfs: rhcos-<version>-live-rootfs.<architecture>.img

    mkdir -p /var/lib/tftpboot/rhcos
    wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-installer-kernel-x86_64
    sudo mv rhcos-installer-kernel-x86_64 /var/lib/tftpboot/rhcos/kernel
    wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-installer-initramfs.x86_64.img
    sudo mv rhcos-installer-initramfs.x86_64.img /var/lib/tftpboot/rhcos/initramfs.img

Apache

    yum -y install httpd
    vim /etc/httpd/conf/httpd.conf
    Listen 80 => Listen 8080
    systemctl enable httpd
    systemctl restart httpd

    mkdir -p /var/www/html/rhcos
    wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-live-rootfs.x86_64.img
    sudo mv rhcos-live-rootfs.x86_64.img /var/www/html/rhcos/rootfs.img

    ansible-playbook tasks/configure_tftp_pxe.yml

Bootstrap node

    cd /var/lib/tftpboot/pxelinux.cfg/
    vi 01-52:54:00:a4:db:5f
 default menu.c32
 prompt 1
 timeout 9
 ONTIMEOUT 1
 menu title ######## PXE Boot Menu ########
 label 1
 menu label ^1) Install Bootstrap Node
 menu default
 kernel rhcos/kernel
 append initrd=rhcos/initramfs.img nomodeset rd.neednet=1 console=tty0 console=ttyS0 ip=dhcp coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.17.100:8080/rhcos/rootfs.img coreos.inst.ignition_url=http://10.1.17.100:8080/ignition/bootstrap.ign   

Master nodes
    cd /var/lib/tftpboot/pxelinux.cfg/
    vi 01-52:54:00:8b:a1:17
    vi 01-52:54:00:ea:8b:9d
    vi 01-52:54:00:f8:87:c7

default menu.c32
 prompt 1
 timeout 9
 ONTIMEOUT 1
 menu title ######## PXE Boot Menu ########
 label 1
 menu label ^1) Install Master Node
 menu default
 kernel rhcos/kernel
 append initrd=rhcos/initramfs.img nomodeset rd.neednet=1 console=tty0 console=ttyS0 ip=dhcp coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.17.100:8080/rhcos/rootfs.img coreos.inst.ignition_url=http://10.1.17.100:8080/ignition/master.ign

Worker nodes
    cd /var/lib/tftpboot/pxelinux.cfg/
    vi 01-52:54:00:31:4a:39
    vi 01-52:54:00:6a:37:32
    vi 01-52:54:00:95:d4:ed

default menu.c32
 prompt 1
 timeout 9
 ONTIMEOUT 1
 menu title ######## PXE Boot Menu ########
 label 1
 menu label ^1) Install Worker Node
 menu default
 kernel rhcos/kernel
 append initrd=rhcos/initramfs.img nomodeset rd.neednet=1 console=tty0 console=ttyS0 ip=dhcp coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.17.100:8080/rhcos/rootfs.img coreos.inst.ignition_url=http://10.1.17.100:8080/ignition/worker.ign


 Step 6: Configure HAProxy as Load balanceron Bastion / Helper node
    yum install -y haproxy
    mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.default
    ansible-playbook tasks/configure_haproxy_lb.yml

Step 7: Install OpenShift installer and CLI binaryon Bastion / Helper node

OpenShift Client binary:

    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
    tar xvf openshift-client-linux.tar.gz
    sudo mv oc kubectl /usr/local/bin
    rm -f README.md LICENSE openshift-client-linux.tar.gz

OpenShift installer binary:

    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz
    tar xvf openshift-install-linux.tar.gz
    sudo mv openshift-install /usr/local/bin
    rm -f README.md LICENSE openshift-install-linux.tar.gz


Check if you can run binaries:

    openshift-install version
    oc version
    kubectl version --client


    ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

Step 8: Generate ignition fileson Bastion / Helper node

    mkdir ~/.openshift
    vim  ~/.openshift/pull-secret
    mkdir -p ~/ocp4
    cd ~/

    cat <<EOF > install-config-base.yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetworks:
  - cidr: 10.254.0.0/16
    hostPrefix: 24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '$(< ~/.openshift/pull-secret)'
sshKey: '$(< ~/.ssh/id_rsa.pub)'
EOF

    cd ~/
    cp install-config-base.yaml ocp4/install-config.yaml
    cd ocp4
    openshift-install create manifests
    sed -i 's/true/false/' manifests/cluster-scheduler-02-config.yml
    openshift-install create ignition-configs
    $ ls
    auth  bootstrap.ign  master.ign  metadata.json  worker.ign
    $ tree
    .
    ├── auth
    │   ├── kubeadmin-password
    │   └── kubeconfig
    ├── bootstrap.ign
    ├── master.ign
    ├── metadata.json
    └── worker.ign

    mkdir -p /var/www/html/ignition
    cp -v *.ign /var/www/html/ignition
    chmod 644 /var/www/html/ignition/*.ign

    ls /var/www/html/ignition/
    bootstrap.ign  master.ign  worker.ign

   systemctl enable --now haproxy.service dhcpd httpd tftp named
   systemctl restart haproxy.service dhcpd httpd tftp named
   systemctl status haproxy.service dhcpd httpd tftp named

Step 9: Create Bootstrap, Masters and Worker VMs(On Hypervisor Node)



