# 
Login
    source /usr/local/etc/ocp4.config
    oc login -u kubeadmin -p ${RHT_OCP4_KUBEADM_PASSWD} https://api.ocp4.example.com:6443
    
Show nodes

    oc get nodes
    oc adm top node
    
    oc describe node master01
    
Show pods    

    oc get pod -n openshift-image-registry
    oc logs --tail 3 -n openshift-image-registry cluster-image-registry-operator-564bd5dd8f-s46bz

    oc get clusteroperators

Displaying the Logs of OpenShift Nodes

    oc adm node-logs -u crio my-node-name


Apply resource quota 
    oc create -f resource-quota-1.yaml -n dev-memsql

Add label to woker24-27

    for i in worker24.cp4d.datalake.vnpt.vn worker25.cp4d.datalake.vnpt.vn worker26.cp4d.datalake.vnpt.vn worker27.cp4d.datalake.vnpt.vn
    do
        oc label node $i tenant=tenant2
    done

    for i in worker24.cp4d.datalake.vnpt.vn worker25.cp4d.datalake.vnpt.vn worker26.cp4d.datalake.vnpt.vn worker27.cp4d.datalake.vnpt.vn
    do
        oc get node $i --show-labels
    done

oc patch namespace dev-memsql -p '{"metadata":{"annotations":{"openshift.io/node-selector":"tenant=tenant2"}}}'