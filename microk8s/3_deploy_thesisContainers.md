# Create a namespace with

`sudo microk8s kubectl create ns neil-test`

# Create a pv  

call it `pv-localk8s.yaml`
```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: neiltest-rstudio
spec:
  capacity:
    storage: 1Gi
  hostPath:
    path: "/var/snap/microk8s/common/default-storage/"
    type: ""
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: microk8s-hostpath
  volumeMode: Filesystem
```
apply with `sudo microk8s kubectl apply -f ./persistentStorage/Rancher/pv-localk8s.yaml`  

# Create a pvc  

call it `pvc-localk8s.yaml`

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: "neiltest-rstudio"
spec:
  storageClassName: "microk8s-hostpath"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  volumeName: "neiltest-rstudio"
```
apply with `sudo microk8s kubectl apply -f ./persistentStorage/Rancher/pvc-localk8s.yaml --namespace neil-test`

# Deploy the helm chart  

From the `Thesis-Helm-Charts` gitHub project with:

`node2@node2:~/gitDocs/Thesis-Helm-Chart$ sudo microk8s helm3 install analyticenv ./analyticEnvironment-0.5.0.tgz --set rstudio.persistentStorage.firstClaimName="neiltest-rstudio" --set mongodb.enabled=false --set postgresql.enabled=false --set rstudio.imageTag="v-1.5" --namespace neil-test`

# Make the service a LoadBalancer Type

`node2@node2:~/gitDocs/Thesis-Helm-Chart$ sudo microk8s kubectl patch svc analytics --namespace neil-test -p '{"spec": {"type": "LoadBalancer"}}'`