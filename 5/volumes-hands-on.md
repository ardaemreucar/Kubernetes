# Kubernetes Hands-on: Using K8s Volumes

Purpose of this hands-on training is to give students the knowledge of Kubernetes Volumes.

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- Explain the need for persistent data management.

- Learn `Persistent Volumes` and `Persistent Volume Claims`.

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Kubernetes Volume Persistence

- Part 3 - Binding PV to PVC

- Part 4 - hostPath and EmptyDir

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 20.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../kubernetes-02-basic-operations/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*



- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Kubernetes Volume Persistence

- Get the documentation of `PersistentVolume` and its fields. Explain the volumes, types of volumes in Kubernetes and how it differs from the Docker volumes. [Volumes in Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/)

```bash
kubectl explain pv
```

- Log into the `kube-worker-1` node, create a `pv-data` directory under home folder, also create an `index.html` file with `Welcome to Kubernetes persistent volume lesson` text and note down path of the `pv-data` folder.

```bash
mkdir pv-data && cd pv-data
echo "Welcome to Kubernetes persistent volume lesson" > index.html
ls
pwd
/home/ubuntu/pv-data
or
for minikube
/tmp/pv-data
```


- Create a `batch107-pv.yaml` file using the following content with the volume type of `hostPath` to build a `PersistentVolume` and explain fields.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: batch107-pv-vol
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/pv-data"
```

- Create the PersistentVolume `batch107-pv-vol`.

```bash
kubectl apply -f batch107-pv.yaml
```

- View information about the `PersistentVolume` and notice that the `PersistentVolume` has a `STATUS` of available which means it has not been bound yet to a `PersistentVolumeClaim`.

```bash
kubectl get pv batch107-pv-vol
```

- Get the documentation of `PersistentVolumeClaim` and its fields.

```bash
kubectl explain pvc
```

- Create a `batch107-pv-claim.yaml` file using the following content to create a `PersistentVolumeClaim` and explain fields.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: batch107-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

- Create the PersistentVolumeClaim `batch107-pv-claim`.

```bash
kubectl apply -f batch107-pv-claim.yaml
```

> After we create the PersistentVolumeClaim, the Kubernetes control plane looks for a PersistentVolume that satisfies the claim's requirements. If the control plane finds a suitable `PersistentVolume` with the same `StorageClass`, it binds the claim to the volume. Look for details at [Persistent Volumes and Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#introduction)

- View information about the `PersistentVolumeClaim` and show that the `PersistentVolumeClaim` is bound to your PersistentVolume `batch107-pv-vol`.

```bash
kubectl get pvc batch107-pv-claim
```

- View information about the `PersistentVolume` and show that the PersistentVolume `STATUS` changed from Available to `Bound`.

```bash
kubectl get pv batch107-pv-vol
```

- Create a `batch107-pod.yaml` file that uses your PersistentVolumeClaim as a volume using the following content.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: batch107-pod
  labels:
    app: batch107-web 
spec:
  volumes:
    - name: batch107-pv-storage
      persistentVolumeClaim:
        claimName: batch107-pv-claim
  containers:
    - name: batch107-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: batch107-pv-storage
```

- Create the Pod `batch107-pod`.

```bash
kubectl apply -f batch107-pod.yaml
```

- Verify that the Pod is running.

```bash
kubectl get pod batch107-pod
```

- Open a shell to the container running in your Pod.

```bash
kubectl exec -it batch107-pod -- /bin/bash
```

- Verify that `nginx` is serving the `index.html` file from the `hostPath` volume.

```bash
curl http://localhost/
```

- Log into the `kube-worker-1` node, change the `index.html`.

```bash
cd pv-data
echo "Kubernetes Rocks!!!!" > index.html
```

- Log into the `kube-master` node, check if the change is in effect.

```bash
kubectl exec -it batch107-pod -- /bin/bash
curl http://localhost/
```

- Expose the batch107-pod pod as a new Kubernetes service on master.

```bash
kubectl expose pod batch107-pod --port=80 --type=NodePort
```

- List the services.

```bash
kubectl get svc
```

- Check the browser (`http://<public-workerNode-ip>:<node-port>`) that batch107-pod is running.

- Delete the `Pod`, the `PersistentVolumeClaim` and the `PersistentVolume`.

```bash
kubectl delete pod batch107-pod
kubectl delete pvc batch107-pv-claim
kubectl delete pv batch107-pv-vol
```

## Part 3 - Binding PV to PVC

- Create a folder name it "pvc-bound"

```bash
mkdir pvc-bound && cd pvc-bound
```

- Create a `pv-3g.yaml` file using the following content with the volume type of `hostPath` to build a `PersistentVolume`.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-3g
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/pv-data"
```

- Create the PersistentVolume `pv-3g`.

```bash
kubectl apply -f pv-3g.yaml
```

- Create a `pv-6g.yaml` file using the following content with the volume type of `hostPath` to build a `PersistentVolume`.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-6g
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 6Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/pv-data"
```

- Create the PersistentVolume `pv-6g`.

```bash
kubectl apply -f pv-6g.yaml
```

- List to PersistentVolume's.

```bash
kubectl get pv
```

- Create a `pv-claim-2g.yaml` file using the following content to create a `PersistentVolumeClaim`.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim-2g
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

- Create the PersistentVolumeClaim `pv-claim-2g`.

```bash
kubectl apply -f pv-claim-2g.yaml
```

- View information about the `PersistentVolumeClaim` and show that the `pv-claim-2g` is bound to PersistentVolume `pv-3g`. Notice that the capacity of the pv-claim-2g is 3Gi.

```bash
kubectl get pvc
```

- Create another PersistentVolumeClaim file and name it `pv-claim-7g.yaml`.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim-7g
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 7Gi
```

- Create the PersistentVolumeClaim `pv-claim-7g`.

```bash
kubectl apply -f pv-claim-7g.yaml
```

- View information about the `PersistentVolume's` and `PersistentVolumeClaim's` and show that the status of `pv-claim-7g` is `pending` and the satus of pv-6g is available. 

```bash
kubectl get pv,pvc
```

- Delete all pv and pvc's.

```bash
kubectl delete -f .
```

## Binding PV to PVC using labels

- Let's create 2 persistent volumes and a persistent volume claim object using the following yaml files.
- Notice that pv1 has 5G of storage, pv2 has 3G of storage and pvc demands 2G storage. Normally 2G will match 3G however there's a selector based on labels. That's why binding works on labels not size.


pv1.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
   name: mysqlpv
   labels:
     app: mysql
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/mysql"
```

pv2.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
   name: mysqlpv2
   labels:
     app: mysql2
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/mysql"
    # originally /home/ubuntu/mysql
```

pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysqlclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  storageClassName: ""
  selector:
    matchLabels:
      app: mysql
```

- after creating files, create objects and observe the result.

```bash
kubectl create -f .
```

```bash
kubectl get pv,pvc
```

## Part 4 - Managing Container Storage with Kubernetes Volumes

## Introduction
Kubernetes volumes offer a simple way to mount external storage to containers. This lab will test your knowledge of volumes as you provide storage to some containers according to a provided specification. This will allow you to practice what you know about using Kubernetes volumes.

## Preparation

Log in to the control plane server with ssh using the credentials provided.

## Task-1: Create a Pod That Outputs Data to the Host Using a Volume

1. Create a Pod that will interact with the host file system by using 

```bash  
maintenance-pod.yml.
```
2. Enter in the first part of the basic YAML for a simple busybox Pod that outputs some data every 5 seconds:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: maintenance-pod
spec:
    containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'while true; do echo Success! >> /output/output.txt; sleep 5; done']
```

3. Under the basic YAML, begin creating volumes, which should be level with the containers spec:

```yaml
volumes:
- name: output-vol
  hostPath: # klasör worker node içerisinde oluşuyor
      path: /var/data
```

4. In the containers spec of the basic YAML, add a line for volume mounts:

```yaml 
volumeMounts:
- name: output-vol
  mountPath: /output # bu da container içindeki klasör
```
5. Save the file.

6. Finish creating the Pod by using 
```bash
kubectl create -f maintenance-pod.yml.
```

7. Make sure the Pod is up and running by using below commands and check that maintenance-pod is running, so it should be outputting data to the host system.

```bash
kubectl get pods
```

8. Log into the ```worker node``` server using the credentials provided.

9.Look at the output by using below command, to see whether the Pod setup was successful.

```bash
cat /var/data/output.txt
```

## Task-2: Create a Multi-Container Pod That Shares Data Between Containers Using a Volume

1. Return to the control plane server.

2. Create another YAML file for a shared-data multi-container Pod by using 

```bash
shared-data-pod.yml.
```
3.  Start with the basic Pod definition and add multiple containers, where the first container will write the **output.txt** file and the second container will read the **output.txt** file:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: shared-data-pod
spec:
    containers:
    - name: busybox1
      image: busybox
      command: ['sh', '-c', 'while true; do echo Success! >> /output/output.txt; sleep 5; done']
    - name: busybox2
      image: busybox
      command: ['sh', '-c', 'while true; do cat /input/output.txt; sleep 5; done']
```
4. Set up the volumes, again at the same level as containers with an emptyDir volume that only exists to share data between two containers in a simple way:

```yaml
volumes:
- name: shared-vol
  emptyDir: {} # bu örnekte pod içerisinde bir klasör oluşuyor
```
5. Mount that volume between the two containers by adding the following lines under command for the busybox1 container:
```yaml
volumeMounts:
- name: shared-vol
  mountPath: /output # busybox1 container içinde
```
6. For the busybox2 container, add the following lines to mount the same volume under command to complete creating the shared file:
```yaml
volumeMounts:
- name: shared-vol
  mountPath: /input # busybox2 içindeki klasör
```

7. Save the file and exit by pressing the ESC key and using **:wq**.

8. Finish creating the multi-container Pod using 

```bash
kubectl create -f shared-data-pod.yml
```
9. Make sure the Pod is up and running by using below command and check if both containers are running and ready.
 
```bash
kubectl get pods
```

10. To make sure the Pod is working, check the logs for shared-data-pod.yml and specify the second container that is reading the data and printing it to the console, using 
```bash
kubectl logs shared-data-pod -c busybox2
```

- entering the container, we can also see the shared folder and file
```bash
kubectl exec -it shared-data-pod -c busybox2 -- sh
ls
```


11. If you see the series of "Success!" messages, you have successfully created both containers, one of which is using a host path volume to write some data to the host disk and the other of which is using an **emptyDir** volume to share a volume between two containers in the same Pod.
    