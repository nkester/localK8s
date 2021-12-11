# Quick Setup  

`cd /var/lib/rancher/rke2/bin/`

`export KUBECONFIG=/etc/rancher/rke2/rke2.yaml`  

`sudo ./kubectl --kubeconfig $KUBECONFIG get pods --all-namespaces`