# localRKE2
These are my notes and files for creating a local Rancher Kubernetes Environment 2 on my Ubuntu desktop.

# Resources

## RKE2 Quickstart  

[RKE2 Quickstart](https://docs.rke2.io/install/quickstart/)  

[gitHub Issue](https://github.com/rancher/rke2/issues/818)
These instructions are what I followed to install RKE2 on my local Ubuntu desktop.

## Helm  

These instructions are from the helm.sh website and are specifically for the apt package manager.
[Helm Quickstart](https://helm.sh/docs/intro/install/#from-apt-debianubuntu)

## Rancher  

Installing Rancher on kubernetes using a helm chart is desribed [here](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/)  


# Starting Conditions  

I have loaded Ubuntu Server 20.04 LTS on two old desktop computers. They are named `node1` and `node2`. I intend for `node1` to serve as the rke2 server and `node2` to serve as the first rke2 agent.

# Setting Up RKE2 Server  

These instructions came from the `RKE2 Quickstart` and `gitHub Issue` referenced above.  

Steps:

1. Ubuntu Server 20.04 LTS with internet  
2. `curl -sfL https://get.rke2.io --output install.sh`  
3. `chmod +x install.sh`  
4. `su` This changes me to the `root` user for install (required)  
5. `INSTALL_RKE2_CHANNEL=stable ./install.sh`  
6. `systemctl enable rke2-server.service`  
7. `systemctl start rke2-server.service`  

In another terminal I followed the logs for this service start using `journalctl -u rke2-server -f`  

The rke2 `install.sh` installs tools like `kubectl` in the following directory but does not place them on the user's path: `/var/lib/rancher/rke2/bin/` for simplicity, I navigated to that directory with `cd /var/lib/rancher/rke2/bin/`

Once complete, check on the status of the pods with the following command:  

`sudo ./kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get pods --all-namespaces`  

To make it easier to point to the `kubeconfig` file, save it as an environment variable with the following command: `export KUBECONFIG=/etc/rancher/rke2/rke2.yaml` then you can use the following command to interact with the cluster:  
`sudo ./kubectl --kubeconfig $KUBECONFIG get pods --all-namespaces`  

# Install Helm  

Now that we have a functioning kubernetes cluster (Rancher Kubernetes Engine 2) lets install `helm` so we can deploy resources into that environment.   

Steps:  
1. `curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -`  
2. `sudo apt-get install apt-transport-https --yes`  
3. `echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list`  
4. `sudo apt-get update`  
5. `sudo apt-get install helm`  

Confirm it installed properly wit the following command: `helm version`  

# Install the Rancher UI

To be clear, the Rancher UI is not the same as RKE (1 or 2). RKE is the underlying kubernetes distribution while the Rancher UI is how we interact with it. Now that we have a working RKE2 instance and have installed `helm`, we will use the Rancher Helm Chart to install the Rancher UI.  

Note, the artifacthub.io page for the stable rancher ui is [here](https://artifacthub.io/packages/helm/rancher-stable/rancher)

## Step 1: Add the Rancher Helm Repo  

`sudo helm repo add rancher-latest https://releases.rancher.com/server-charts/latest`  

## Step 2: Installing a Certificate Manager  

This is required in order to use TLS certificates provided by Rancher. Instructions to do this are provided [here](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/#4-install-cert-manager).  

The certificate manager is installed into our kubernetes cluster.  

First, install the cert manager with:  

`node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml`  

Then add the Jetstack Helm repository with:  
`sudo helm repo add jetstack https://charts.jetstack.io`  

Then update the helm repo with:  
`sudo helm repo update`  

Finally, install the cert-manager Helm Chart with:  
`sudo helm install cert-manager jetstack/cert-manager   --namespace cert-manager   --create-namespace   --version v1.5.1 --kubeconfig $KUBECONFIG`  

Note that we must include the `--kubeconfig` flag to tell helm how to reach RKE2.

Check it installed properly by checking the `cert-manager` namespace with:  

`sudo ./kubectl --kubeconfig $KUBECONFIG get pods --namespace cert-manager`

## Step 3: Create a Kubernetes Namespace for Rancher  

This is where all of the Rancher kubernetes resources will be located. We will call it `cattle-system`.  

Do this with the following command. Note the path we are executing this from:  

`node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG create namespace cattle-system`  

Confirm the namespace was created with: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get ns`

## Step 4: Install Rancher with Helm  

Note that the `hostname` is the name of the node. You can get this by running the following command: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get nodes`

`node1@node1:/var/lib/rancher/rke2/bin$ sudo helm install rancher rancher-latest/rancher --kubeconfig $KUBECONFIG --namespace cattle-system --set bootstrapPassword=admin --set ingress.tls.source=rancher --set hostname=node1`

You can check on the status of the deployment with the following command:
`sudo ./kubectl --kubeconfig $KUBECONFIG get pods --namespace cattle-system`

This can take a long time as each resource must request certificates and be configured. 

At this point I have not set up a load balancer. In order to access the rancher pod from outside the cluster I had to change the service type of the `rancher` service in the `cattle-system` namespace from `ClusterIP` to `NodePort`.   
I did this with the command: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG -n cattle-system edit svc/rancher` at which time I changed the type to `NodePort` and changed the `targetPort` to `nodePort`, changing them from `80` and `443` to `30680` and `30444` respectively.  

Once doing this and applying the change I could access the rancher dashboard from `<host IP>:30444` which translated to `192.168.1.32:30444` on my internal network. Note that I can only access from Chrome. FireFox gives an error.

My `admin` user's password was changed to `QkpDWVzCnhWag8iG`  


## Step 4: Install MetalLB