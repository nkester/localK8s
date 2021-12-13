# Installing Rancher UI on MicroK8s  

This assumes you've completed the steps in `1_setup_microk8s`  

# References  

[Install Rancher on Microk8s - The FN Fast Way in 2022](https://medium.com/donpublic/install-ranch-on-microk8s-the-fn-fast-way-a2f28ea07685)  
*Hopefully this is the only resource needed!*  

# Step 0: Set the Conditions  

Note that I opted to not make commands executable by non-sudo users.

## Priviledged Containers  

Allow priviledged containers to run on the cluster:  
`node1@node1:~$ sudo sh -c 'echo "--allow-priviledged=true" /var/snap/microk8s/current/args/kube-apiserver'`  

# Step 1: Install a Certificate Manager  

This is required in order to use TLS certificates provided by Rancher. Instructions to do this are provided [here](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/#4-install-cert-manager).  

The certificate manager is installed into our kubernetes cluster.  

## (NO LONGER USED) Custom Resource Definitions (CRD)  

First, install required CRDs for the cert manager with:  

`node1@node1:~$ sudo microk8s kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml`  

## Helm Repo  

Add the Jetstack Helm repository with:  

`node1@node1:~$ sudo microk8s helm3 repo add jetstack https://charts.jetstack.io`  

Then update the helm repo with:  

`node1@node1:~$ sudo microk8s helm3 repo update`  

Finally, install the certificate manager. Helm should create the `cert-manager` namespace.

`node1@node1:~$ sudo microk8s helm3 install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.6.1 --set installCRDs=true`  

**Note that I am no longer installing the Custom Resource Definitions on their own. I'm allowing helm to manage these!**  

Confirm the pods are running as expected with `sudo microk8s kubectl get pods --namespace cert-manager`  



```
node1@node1:~$ sudo microk8s helm3 install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.5.1
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /var/snap/microk8s/2695/credentials/client.config
NAME: cert-manager
LAST DEPLOYED: Mon Dec 13 03:36:52 2021
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.5.1 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```

# Step 2: Install Rancher on MicroK8s  

Create a namespace for Rancher:  

`node1@node1:~$ sudo microk8s kubectl create ns cattle-system`  

Add the helm repo and update it:  

`node1@node1:~$ sudo microk8s helm3 repo add rancher-latest https://releases.rancher.com/server-charts/latest`  

`node1@node1:~$ sudo microk8s helm3 repo update`  


