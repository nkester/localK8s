# localRKE2  
These are my notes and files for creating a local Rancher Kubernetes Environment 2 on my Ubuntu desktop.  

**TAKE AWAY:** This guide will be good for the future but I believe my old desktops do not have the power to support this deployment. I'm going to try again with microk8s!

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

## Ceph (File, Block, and Object Storage)  

[Installing CEPH](https://docs.ceph.com/en/pacific/install/)  

[Rook.io](https://rook.io/) - Open-source, Cloud-Native Storage for Kubernetes.   

## Longhorn (Block Storage)  

[Longhorn GitHub](https://github.com/longhorn/longhorn)  

[Longhorn Documentation](https://longhorn.io/docs/1.2.2/deploy/install/#installation-requirements)

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

This helm chart creates a rancher service (`service/rancher`) as a ClusterIP. Just run this command to patch that service and make it a LoadBalancer type service instead of a ClusterIP.  

`sudo ./kubectl --kubeconfig $KUBECONFIG patch svc rancher -p '{"spec": {"type": "LoadBalancer"}}' --namespace cattle-system`


My `admin` user's password was changed to `QkpDWVzCnhWag8iG`  

## Step 6: Setting up Storage  

Based on the desire to build the most robust possible distribution to limit future rework/rebuild, I've opted to try a Ceph install as this support File, Block, and Object Storage.  

According to the ceph docs website, there are two methods to install ceph: `cephadm` and `rook`. Based on the suggestion in the *Domesticating Kubernetes* article, I've decided to try `rook` first.  

**NOTE: CEPH WAS TO HARD AT THE MOMENT. I DECIDED TO GO WITH LONGHORN TO CONTINUE MOVING**

### Rook.io  

Accoring to rook.io  

> "Rook turns distributed storage systems into self-managing, self-scaling, self-healing storage services. It automates the tasks of a storage administrator: deployment, bootstrapping, configuration, provisioning, scaling, upgrading, migrating, disaster recovery, monitoring, and resource management."  

** ISSUE **: Ceph requires raw partions. I do not have these yet and in order to create a new partion from my drive I'll need to re-install everything. I may set up a ceph cluster on `node2` for testing and then rebuild everything later.  

This is a helpful resource [here](https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux) on how to create new partions.  

** This may be the solution ** This [ask ubunut](https://askubuntu.com/questions/126153/how-to-resize-partitions) talks about using a live CD to accomplish this.

#### My Steps to Partition a Drive  

From the Digital Ocean article.  

Starting condition:  

```
node1@node1:~$ sudo lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0  55.4M  1 loop /snap/core18/2128
loop1                       7:1    0  61.9M  1 loop /snap/core20/1242
loop2                       7:2    0  55.5M  1 loop /snap/core18/2253
loop3                       7:3    0  32.3M  1 loop /snap/snapd/12704
loop4                       7:4    0  70.3M  1 loop /snap/lxd/21029
loop5                       7:5    0  42.2M  1 loop /snap/snapd/14066
loop6                       7:6    0  67.2M  1 loop /snap/lxd/21835
sda                         8:0    0 465.8G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     1G  0 part /boot
└─sda3                      8:3    0 464.8G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   200G  0 lvm  /
sdb                         8:16   0 465.7G  0 disk
└─sdb1                      8:17   0 465.7G  0 part
sr0                        11:0    1  1024M  0 rom
```
Make a new label using the `gpt` method.

`sudo parted /dev/sdb mklabel gpt` Where `/dev/sdb` was my attached USB HDD.  

This will ask if you are sure you want to do this and erase all data.  

Ending condition: 

```
node1@node1:~$ sudo lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0  55.4M  1 loop /snap/core18/2128
loop1                       7:1    0  61.9M  1 loop /snap/core20/1242
loop2                       7:2    0  55.5M  1 loop /snap/core18/2253
loop3                       7:3    0  32.3M  1 loop /snap/snapd/12704
loop4                       7:4    0  70.3M  1 loop /snap/lxd/21029
loop5                       7:5    0  42.2M  1 loop /snap/snapd/14066
loop6                       7:6    0  67.2M  1 loop /snap/lxd/21835
sda                         8:0    0 465.8G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     1G  0 part /boot
└─sda3                      8:3    0 464.8G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   200G  0 lvm  /
sdb                         8:16   0 465.7G  0 disk
sr0                        11:0    1  1024M  0 rom
```
Next step is to create a partition of the entire disk with:  

`node1@node1:~$ sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%`  

The result:  

```
node1@node1:~$ sudo lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0  55.4M  1 loop /snap/core18/2128
loop1                       7:1    0  61.9M  1 loop /snap/core20/1242
loop2                       7:2    0  55.5M  1 loop /snap/core18/2253
loop3                       7:3    0  32.3M  1 loop /snap/snapd/12704
loop4                       7:4    0  70.3M  1 loop /snap/lxd/21029
loop5                       7:5    0  42.2M  1 loop /snap/snapd/14066
loop6                       7:6    0  67.2M  1 loop /snap/lxd/21835
sda                         8:0    0 465.8G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     1G  0 part /boot
└─sda3                      8:3    0 464.8G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   200G  0 lvm  /
sdb                         8:16   0 465.7G  0 disk
└─sdb1                      8:17   0 465.7G  0 part
sr0                        11:0    1  1024M  0 rom
```  

```
node1@node1:~$ sudo lsblk -f
NAME                    FSTYPE      LABEL       UUID                                   FSAVAIL FSUSE% MOUNTPOINT
loop0                   squashfs                                                             0   100% /snap/core18/2128
loop1                   squashfs                                                             0   100% /snap/core20/1242
loop2                   squashfs                                                             0   100% /snap/core18/2253
loop3                   squashfs                                                             0   100% /snap/snapd/12704
loop4                   squashfs                                                             0   100% /snap/lxd/21029
loop5                   squashfs                                                             0   100% /snap/snapd/14066
loop6                   squashfs                                                             0   100% /snap/lxd/21835
sda
├─sda1
├─sda2                  ext4                    6b9040c5-0e91-4cf7-8ff6-6e71f468cc9d    706.1M    21% /boot
└─sda3                  LVM2_member             QrhSFy-JbWY-ROJ4-Jj3r-X52c-hTlU-cXB4qQ
  └─ubuntu--vg-ubuntu--lv
                        ext4                    14438489-d861-43a7-98c8-ed0ac77ca109    168.8G     9% /
sdb
└─sdb1                  ntfs        My Passport 24AE59CBAE5995E0
sr0
```
**THIS IS NOT DONE, HOW DO YOU CREATE A "RAW" OR "EMPTY" DRIVE WITH NO FILE SYSTEM??**  

#### Install the Ceph Operator using Helm  

Add the helm repo: `node1@node1:/var/lib/rancher/rke2/bin$ sudo helm repo add rook-release https://charts.rook.io/release`  

Update the repos: `node1@node1:/var/lib/rancher/rke2/bin$ sudo helm repo update`  

Install the chart: `node1@node1:/var/lib/rancher/rke2/bin$ sudo helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph --kubeconfig $KUBECONFIG`

### Longhorn  

The documentation for longhorn is FAR better than ceph and rook...it assumes less of me which is good.  

Before starting, I checked the installation requirements [here](https://longhorn.io/docs/1.2.2/deploy/install/#installation-requirements). The Longhorn team is nice enough to provide a script that checks all of these conditions for us on this page. I'll check that first!

The first thing I did was finally add the `rke2/bin` directory to my path using `export PATH=$PATH:/var/lib/rancher/rke2/bin/`  

Then, running the checklist with this `curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.2.2/scripts/environment_check.sh | bash` told me that I needed to install `jq`  

I did this with the following commands:  

```
sudo apt update
sudo apt -y upgrade
sudo apt -y intall jq
```  
Once that was installed, I ran the checklist script again and got a positive response (I'm not sure what the cleanup error means)  

```
node1@node1:~$ curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.2.2/scripts/environment_check.sh | bash
The connection to the server localhost:8080 was refused - did you specify the right host or port?
The connection to the server localhost:8080 was refused - did you specify the right host or port?
all pods ready (/)
The connection to the server localhost:8080 was refused - did you specify the right host or port?
The connection to the server localhost:8080 was refused - did you specify the right host or port?

  MountPropagation is enabled!

cleaning up...
error: unable to recognize "/tmp/tmp.CFVwaKOGvg/environment_check.yaml": Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
clean up complete
```  

Next, following the conditions check documentation referenced above, I installed `open-iscsi` and the `iscsi` installer with the following commands:  

`sudo apt-get install open-iscsi` and `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.2.2/deploy/prerequisite/longhorn-iscsi-installation.yaml` which I then checked for successful deployment with `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get pod | grep longhorn-iscsi-installation`  

That that ran successfully I installed `nfsv4` and the `nfs` installer with:  

`sudo apt-get -y install nfs-common` and `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.2.2/deploy/prerequisite/longhorn-nfs-installation.yaml` which I then checked for successful deployment with `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get pod | grep longhorn-nfs-installation`  

#### Install Longhorn  

Now with the requirements met, let's install Longhorn using helm.  The instructions are [here](https://longhorn.io/docs/1.2.2/deploy/install/install-with-helm/)  

  1) Add the repo: `node1@node1:/var/lib/rancher/rke2/bin$ sudo helm repo add longhorn https://charts.longhorn.io`  
  2) Update the repo: `node1@node1:/var/lib/rancher/rke2/bin$ sudo helm repo update`  
  3) Create the longhorn-system namespace: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG create ns longhorn-system`  
  4) Install the chart: `node1@node1:/var/lib/rancher/rke2/bin$ sudo helm install longhorn longhorn/longhorn --namespace longhorn-system --kubeconfig $KUBECONFIG`  
  5) Check the status with: `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get pods --namespace longhorn-system`  
  
#### Install Ingress for Rancher UI  

  1)  `node1@node1:~$ echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth`  
  2)  `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG -n longhorn-system create secret generic basic-auth --from-file=$HOME/auth`  
  3)   Change the ClusterIP Service to a LoadBalancer service for the longhorn-frontend service with `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG patch svc longhorn-frontend -p '{"spec": {"type": "LoadBalancer"}}' --namespace longhorn-system`  
  4)  Identify the LoadBalancer IP to access the Longhorn UI with `node1@node1:/var/lib/rancher/rke2/bin$ sudo ./kubectl --kubeconfig $KUBECONFIG get svc -n longhorn-system`  
  
#### Adding Additional Disks  

I had an external hard drive that I wanted to use for this. I plugged it into `node1` and performed the following steps to format and partition it.  

This is referenced in the following resources [here-best](https://www.tecmint.com/create-disk-partitions-in-linux/) and [here](https://www.tecmint.com/create-new-ext4-file-system-partition-in-linux/)  

  1) Use the `parted` tool to list available drives with `sudo parted -l`  
  2) Once you identify the drive you want, enter the `parted` tool with `sudo parted /dev/sdb` (or whatever your drive is)  
  3) Then create a label, `msdos` worked for me. `(parted) mklabel msdos`. Note that I also tried `gpt` but that did not work. I think the issue was another step in the process, however.  
  4) Next create a partition. First I tried to use the entire drive by specifing `0% 100%` but that did not work. Instead, I made two partitions. The first small, with this `(parted) mkpart primary ext4 0 1GB`. Ensure it made the partition with `(parted) print`.  
  5) Next, make the second partition. My drive was 500GB large. My second partition looked like `(parted) mkpart primary ext4 1GB 500GB`.  
  6) Next, quit the `parted` tool with `(parted) quit`  
  7) Now we need to put a file system on these partitions. Do so with the following commands: `sudo mkfs.ext4 /dev/sdb1` and `sudo mkfs.ext4 /dev/sdb2`  
  8) Now that we have partitions and filesystems, we need to create mount points and then mount them. Do so with the following:  
  ```
  sudo mkdir -p /mnt/sdb1
  sudo mkdir -p /mnt/sdb2
  sudo mount -t auto /dev/sdb1 /mnt/sdb1
  sudo mount -t auto /dev/sdb2 /mnt/sdb2 
  ```
  Ensure they have mounted with `df -hT` and `sudo lsbft -f`  
  
  Once they are mounted, go to the Lonhorn UI, like the `Nodes` tab, and then the drop down `Edit node and disks` for `node1`.  Now, add a disk, provide the mount point and a name.  
  
My longhorn IP is: http://192.168.1.152