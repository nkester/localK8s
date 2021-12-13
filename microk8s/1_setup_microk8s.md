# microk8s  

These are my notes on deploying a microk8s environment on my local Ubuntu Desktops.  

# References  

## General  

A great article on setting up a home kubernetes cluster. It is called [Domesticating Kubernetes](https://blog.quickbird.uk/domesticating-kubernetes-d49c178ebc41?source=collection_home---3------0-----------------------)  

An article about installing the Rancher UI on microk8s [Install Rancher on MicroK8s - The FN Fast Way in 2022](https://medium.com/donpublic/install-ranch-on-microk8s-the-fn-fast-way-a2f28ea07685)  

Straight from Ubuntu.com [Installing a local Kubernetes with MircoK8s](https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#1-overview)  

## Logging/Monitoring  

[Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)  

[Grafana Documentation](https://grafana.com/docs/grafana/latest/)  

## Miscellaneous  

[microk8s Ingress addon](https://microk8s.io/docs/addon-ingress)  

[microk8s Registry addon](https://microk8s.io/docs/registry-built-in)  
  * I may not keep this. There is no UI and there seems to be some constraints. I may look at transitioning to a different FOSS solution.* 


# Initial Conditions  

I've started with an old Desktop machine (cira 2010) with Ubuntu Server 20.04 LTS installed.  

# Install MicroK8s  

This follows the steps found in the [Installing a local Kubernetes with MircoK8s](https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#1-overview) referenced above.  

Start by installing microK8s with `sudo snap install microk8s --classic`  

The following is a successful response `microk8s (1.22/stable) v1.22.4 from Canonical✓ installed`  

The following commands allow pod-to-pod and pod-to-internet communication:  

`sudo ufw allow in on cni0 && sudo ufw allow out on cni0` and `sudo ufw default allow routed` Note that `ufw` is the ubuntu firewall.  

Interact with the cluster in the normal way through `kubectl` with the following: `sudo microk8s kubectl <command>`
  
Also, because I had issues with resources when installing `rke2` I want to track how much I'm using as we go through the process. At this point, there is what I'm using:  

CPU:
```
node1@node1:~$ sudo lscpu | grep "MHz"
CPU MHz:                         1700.000
CPU max MHz:                     2900.0000
CPU min MHz:                     800.0000
```  

Memory:  
```
node1@node1:~$ free -h
              total        used        free      shared  buff/cache   available
Mem:          3.6Gi       789Mi       782Mi       1.0Mi       2.1Gi       2.6Gi
Swap:         3.6Gi          0B       3.6Gi
```
   
## Enable microk8s addons  

MicroK8s comes, by default, as a bare bones distribution. Lets add some more capability. Here is a list of available important addons:  
  
  * dns: Deploy DNS. This addon may be required by others, thus we recommend you always enable it.  
  * dashboard: Deploy kubernetes dashboard.  
  * storage: Create a default storage class. This storage class makes use of the hostpath-provisioner pointing to a directory on the host.  
  * ingress: Create an ingress controller.  
  * gpu: Expose GPU(s) to MicroK8s by enabling the nvidia-docker runtime and nvidia-device-plugin-daemonset. Requires NVIDIA drivers to be already installed on the host system.  
  * istio: Deploy the core Istio services. You can use the microk8s istioctl command to manage your deployments.  
  * registry: Deploy a docker private registry and expose it on localhost:32000. The storage addon will be enabled as part of this addon.  

From this list we will enable: `dns`, `dashboard`, `storage`,`ingress`, `istio`, and `registry`  

Do this with the following command `node1@node1:~$ sudo microk8s enable dns dashboard storage ingress istio registry helm3 metallb prometheus`  

When asked for the `metallb` IP range, I gave the following: `192.168.1.150-192.168.1.254`  

By default, the dashboard is exposed via a ClusterIP service. Change it to a LoadBalancer IP with the following:  

`node1@node1:~$ sudo microk8s kubectl patch svc kubernetes-dashboard --namespace kube-system -p '{"spec": {"type": "LoadBalancer"}}'`  

Find the assigned IP with: `sudo microk8s kubectl get svc -n kube-system`. Next, get the token to log into the dashboard that the provide External IP with `sudo microk8s dashboard-proxy`.  Provide the access token to the dashboard to login. This token will time out so run the command above to get a new one.  

### Accessing Prometheus Dashboard  

[Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)

One of the addons we enabled was the Prometheus logging service. By default, this tool is exposed through a ClusterIP service. Of course, this does not help us. Like with the microk8s dashboard, since we also enabled metallb as a local load balancer, lets change this prometheus service to type `LoadBalancer` so we can interact with it.   

By default, Prometheus is installed into the `monitoring` namespace. Check out all of the services in this namespace with: `node1@node1:~$ sudo microk8s  kubectl get svc -n monitoring`  

While there are several services associated with Prometheus, the one we want to expose is `prometheus-k8s`. Do this with the following command.    

`node1@node1:~$ sudo microk8s kubectl patch svc prometheus-k8s --namespace monitoring -p '{"spec": {"type": "LoadBalancer"}}'`

No, get the services in the `monitoring` namespace again and go to the new External IP. Note that you need to specify port `9090`.  

I'll learn and document using Prometheus in a different project. For starters, the following PromQL Queries seemed immediately useful to me.  
  
  * Memory usage over time: `:node_memory_MemAvailable_bytes:sum`  
  * Compute usage over time: `cluster:node_cpu:sum_rate5m`  
  * The number of running containers over time: `kubelet_running_containers{container_state="running"}`  

I added these graphs as their own panel.

### Accessing Grafana Dashboard  

[Grafana Documentation](https://grafana.com/docs/grafana/latest/)  

Grafana is another great logging and metrics tool.  

Use the same approach to expose this service to the load balancer for an external IP with:  

`node1@node1:~$ sudo microk8s kubectl patch svc grafana --namespace monitoring -p '{"spec": {"type": "LoadBalancer"}}'`  

And then access it using the provided external IP appended with port `3000`  

The default login is `admin` `admin`. 

It is super easy to connect grafana to the prometheus logs immediately. Do this by providing grafana with the `prometheus-k8s` service's ClusterIP and port `9090`.   

Like prometheus, I'll research and document this in another project.  

