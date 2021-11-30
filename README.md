# localRKE2
These are my notes and files for creating a local Rancher Kubernetes Environment 2 on my Ubuntu desktop.

# Resources

## General

A great article on setting up a home kubernetes cluster. It is called [Domesticating Kubernetes](https://blog.quickbird.uk/domesticating-kubernetes-d49c178ebc41?source=collection_home---3------0-----------------------)

## RKE2 Quickstart  

[RKE2 Quickstart](https://docs.rke2.io/install/quickstart/)  

[gitHub Issue](https://github.com/rancher/rke2/issues/818)
These instructions are what I followed to install RKE2 on my local Ubuntu desktop.

## Helm  

These instructions are from the helm.sh website and are specifically for the apt package manager.
[Helm Quickstart](https://helm.sh/docs/intro/install/#from-apt-debianubuntu)

## Rancher  

Installing Rancher on kubernetes using a helm chart is desribed [here](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/)  

## MetalLB  

To provide a load balancing service in my home lab I will use MetalLB. These are resources I used to set up this service.  

[MetalLB Website](https://metallb.universe.tf/)  
* Note, according to the MetalLB website, this is considered a beta tool so setup may change from what is written here!*  

[Installing MetalLB on a home network](https://opensource.com/article/20/7/homelab-metallb)  

[vZilla Kubernetes Playground](https://vzilla.co.uk/vzilla-blog/building-the-home-lab-kubernetes-playground-part-7) 
* Note, this is a great resource and much more clear than others.*

# Starting Conditions  

I have loaded Ubuntu Server 20.04 LTS on two old desktop computers. They are named `node1` and `node2`. I intend for `node1` to serve as the rke2 server and `node2` to serve as the first rke2 agent.

In my home router I set static IP addresses for `node1` and `node2`.  

I also turned off `swap` in accordance with instructions from the "Domesticating Kubernetes" article under the "OS Setup" section. 

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

## Step 3: Install MetalLB  

Based on the **opensource.com** and **vZilla** articles referenced above, I will use the "Address Resolution Protocol" (ARP) method to setup the load balancer.  

First, add the MetalLB helm repository with `$ sudo helm repo add metallb https://metallb.github.io/metallb` and then update the repo with `$ sudo helm repo update`.

This approach is based on a combination of the opensource.com and vZilla articles. I want to deploy this resource as a helm chart because that allows us to easily manage all of the resources created for this technology. This load balancer, specifically, install at least four resources outside the namespace (two each cluster role bindings and cluster roles). The issue, however, is that the helm chart does not properly create the config map. The solution is that we will manage the config map manually and tell the helm chart where to find it. 

The first step is to create a namespace for the metallb system. Do this with:
`node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG create ns metallb-system`

Then create the configmap. Here is the metallbConfigmap.yaml I used:  

---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
      - name: default
        protocol: layer2
        addresses:
          - 192.168.1.150-192.168.1.254
---

Apply that to the namespace with: 
`node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG apply -f ~/metallbConfigmap.yaml --namespace metallb-system`  

Then create a simple yaml with this ConfigMap's name to pass to the helm chart. In this case I called it `metallbValues.yaml` and it consisted of only the following line:  

---
existingConfigMap: config
---

Then I installed MetalLB with `$ sudo helm install metallb-system metallb/metallb -f ~/metallbValues.yaml --kubeconfig $KUBECONFIG --namespace metallb-system`

### Confirm It Works!  

Once that helm chart has deployed, confirm it works by creating another namespace and applying the following simple raw manifest.  

`node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG create ns metallb-test`

I called this yaml `metallb-test-deploy.yaml`:  

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  labels:
    app: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
---
kind: Service
apiVersion: v1
metadata:
  name: hello
  labels:
    app: hello
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
  selector:
    app: hello
  type: LoadBalancer
  externalTrafficPolicy: Cluster
---

Apply this raw manifest with `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG apply -f ~/metallb-test-deploy.yaml --namespace metallb-test`

Now, when getting all resources from that test namespace with `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get all --namespace metallb-test` you should see that your `service/hello` has an External IP assigned to it. It should be the first in the range you provided the metallb-system. If it shows `<pending>` then something went wrong!

```
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
service/hello   LoadBalancer   10.43.31.246   192.168.1.150   80:31760/TCP   39m
```

Visit the External IP on a computer on your home network but not on `node1`. You should see an NGINX message.

### Another attempt

THIS ONE WORKED!
[vZilla Kubernetes Playground](https://vzilla.co.uk/vzilla-blog/building-the-home-lab-kubernetes-playground-part-7)  

This range of IP addresses worked. If this works for the helm chart I'll reconfig my router to not include these IP addresses. `192.168.1.150-192.169.1.254`

** The issue with this approach is that it is difficult to remove resources when they are not managed by helm. I have included a working soultion above.**

This is the command that worked to create the secret
`node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl create secret generic memberlist --kubeconfig $KUBECONFIG -n metallb-system --from-literal=secretkey="$(openssl rand -base64 128)"`

## Step 4: Create a Kubernetes Namespace for Rancher  

This is where all of the Rancher kubernetes resources will be located. We will call it `cattle-system`.  

Do this with the following command. Note the path we are executing this from:  

`node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG create namespace cattle-system`  

Confirm the namespace was created with: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get ns`

## Step 5: Install Rancher with Helm  

Note that the `hostname` is the name of the node. You can get this by running the following command: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get nodes`

`node1@node1:/var/lib/rancher/rke2/bin$ sudo helm install rancher rancher-latest/rancher --kubeconfig $KUBECONFIG --namespace cattle-system --set bootstrapPassword=admin --set ingress.tls.source=rancher --set hostname=node1`

You can check on the status of the deployment with the following command:
`sudo ./kubectl --kubeconfig $KUBECONFIG get pods --namespace cattle-system`

This can take a long time as each resource must request certificates and be configured. 

At this point I have not set up a load balancer. In order to access the rancher pod from outside the cluster I had to change the service type of the `rancher` service in the `cattle-system` namespace from `ClusterIP` to `NodePort`.   
I did this with the command: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG -n cattle-system edit svc/rancher` at which time I changed the type to `NodePort` and changed the `targetPort` to `nodePort`, changing them from `80` and `443` to `30680` and `30444` respectively.  

Once doing this and applying the change I could access the rancher dashboard from `<host IP>:30444` which translated to `192.168.1.32:30444` on my internal network. Note that I can only access from Chrome. FireFox gives an error.

My `admin` user's password was changed to `QkpDWVzCnhWag8iG`  



