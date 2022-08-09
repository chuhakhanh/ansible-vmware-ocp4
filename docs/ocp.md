

Export

    rhcos_ver=4.10.16
    export KUBECONFIG=/root/ocp4upi/"$rhcos_ver"/auth/kubeconfig

Verify that there are not pods in Failed status

    oc get pods --all-namespaces | grep -v -E 'Running|Completed'