
#
    rhcos_ver=4.10.16
    export KUBECONFIG=/root/ocp4upi/"$rhcos_ver"/auth/kubeconfig

Verify that there are not pods in Failed status

    oc get pods --all-namespaces | grep -v -E 'Running|Completed'

### static pod

    [root@master01 manifests]# ll /etc/kubernetes/manifests/
    total 60
    -rw-r--r--. 1 root root 23927 Aug  8  2022 etcd-pod.yaml
    -rw-r--r--. 1 root root   265 Oct  5 09:38 haproxy.yaml
    -rw-r--r--. 1 root root  9262 Jan 23 02:14 kube-apiserver-pod.yaml
    -rw-r--r--. 1 root root  8532 Aug  8  2022 kube-controller-manager-pod.yaml
    -rw-r--r--. 1 root root  4990 Aug  8  2022 kube-scheduler-pod.yaml

    [root@master01 kubelet.service.d]# ll /etc/systemd/system/kubelet.service.d/
    total 12
    -rw-r--r--. 1 root root  0 Aug  8  2022 10-mco-default-env.conf
    -rw-r--r--. 1 root root 62 Aug  8  2022 10-mco-default-madv.conf
    -rw-r--r--. 1 root root 44 Aug  8  2022 20-logging.conf
    -rw-r--r--. 1 root root 87 Feb 10 10:52 20-nodenet.conf


### 
    journalctl -xeu kubelet
    journalctl -xf -u kubelet -n 1000

On node

    sudo crictl ps -a
    

