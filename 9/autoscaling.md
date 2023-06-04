# Kubernetes Hands-on Day-6 :  Autoscaling

Purpose of the this hands-on training is to give students the knowledge of  Autoscaling.


## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 20.04 with two nodes (one master, one worker)
- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```


### Deploy the application

- Create a `php-apache` directory and change directory.

```bash
mkdir php-apache
cd php-apache
```

- Create a `php-apache.yaml` file for second application.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 500Mi
            cpu: 100m
          requests:
            memory: 250Mi
            cpu: 80m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache-service
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
    nodePort: 30002
  selector:
    run: php-apache 
  type: NodePort	
```

Note how the `Deployment` and `Service` `yaml` files are merged in one file. 

Deploy this `php-apache` file.

```bash
kubectl apply -f . 
```

Get the pods.

```bash
kubectl get po
```

Get the services.

```bash
kubectl get svc
```

- Let's check what web app presents us.

- On opening browser (http://<public-node-ip>:<node-port>) we see

```text
OK!
```

- Alternatively, you can use;
```text
curl <public-worker node-ip>:<node-port>
OK!
```

- Do not forget to open the Port <node-port> in the security group of your node instance. 

## Part 4 - Autoscaling in Kubernetes

### Benefits of Autoscaling
To understand better where autoscaling would provide the most value, let’s start with an example. Imagine you have a 24/7 production service with a load that is variable in time, where it is very busy during the day in the US, and relatively low at night. Ideally, we would want the number of nodes in the cluster and the number of pods in deployment to dynamically adjust to the load to meet end user demand. The new Cluster Autoscaling feature together with Horizontal Pod Autoscaler can handle this for you automatically.

### Run & expose php-apache server  

- First, let's check the php-apache `Services` and `Pods` to see if they are still running.

- Observe pods and services:

```bash
kubectl get po

kubectl get svc
```

- Add `watch` board to verify the latest status of Cluster by below Commands.(This is Optional as not impacting the Functionality of Cluster). Observe in a separate terminal.

```bash
$ kubectl get service,hpa,pod -o wide
$ watch -n1 !!
```

### Create Horizontal Pod Autoscaler   

- Now that the server is running, we will create the autoscaler using kubectl autoscale. The following command will create a Horizontal Pod Autoscaler that maintains between 2 and 10 replicas of the Pods controlled by the php-apache deployment we created in the first step of these instructions. Roughly speaking, HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all Pods of 50% (since each pod requests 200 milli-cores by kubectl run), this means average CPU usage of 100 milli-cores). See [here]( https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details ) for more details on the algorithm.

Now activate the  HPAs; 


```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=2 --max=10 
```
or we can use yaml files.

```bash
cd
mkdir auto-scaling && cd auto-scaling
cat << EOF > hpa-php-apache.yaml

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50

EOF
```

```

```bash
$ kubectl apply -f hpa-php-apache.yaml
```

Let's look at the status:
```bash
watch -n3 kubectl get service,hpa,pod -o wide 
```
- php-apache Pod number increased to 2, minimum number.  
- The HPA line under TARGETS shows `<unknown>/50%`. The `unknown` means the HPA can't idendify the current use of CPU.


We may check the current status of autoscaler by running:  
```bash
kubectl get hpa
```

```bash
$ kubectl describe hpa
```

- The `metrics` can't be calculated. So, the `metrics server` should be uploaded to the cluster.

### Install Metric Server 

- First Delete the existing Metric Server if any.

```bash
$ kubectl delete -n kube-system deployments.apps metrics-server
```

- Get the Metric Server form [GitHub](https://github.com/kubernetes-sigs/metrics-server/releases/tag/v0.5.0).

```bash
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
```

- Edit the file `components.yaml`. You will select the `Deployment` part in the file. Add the below line to `containers.args field under the deployment object`.

```yaml
        - --kubelet-insecure-tls
``` 
(We have already done for this lesson)

```yaml
apiVersion: apps/v1
kind: Deployment
......
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
......	
```

- Add `metrics-server` to your Kubernetes instance.

```bash
$ kubectl apply -f components.yaml
```
- Wait 1-2 minute or so.

- Verify the existace of `metrics-server` run by below command

```bash
$ kubectl -n kube-system get pods
```

- Verify `metrics-server` can access resources of the pods and nodes.

```bash
$ kubectl top pods
```
- Look at the the values under TARGETS. The values are changed from `<unknown>/50%` to `1%/50%` & `2%/50%`, means the HPA can now idendify the current use of CPU.

- If it is still `<unknown>/50%`, check the `spec.template.spec.containers.resources.request` field of deployment.yaml files. It is required to define this field. Otherwise, the autoscaler will not take any action for that metric. 

> For per-pod resource metrics (like CPU), the controller fetches the metrics from the resource metrics API for each Pod targeted by the HorizontalPodAutoscaler. Then, if a target utilization value is set, the controller calculates the utilization value as a percentage of the equivalent resource request on the containers in each Pod.
 
> Please note that if some of the Pod's containers do not have the relevant resource request set, CPU utilization for the Pod will not be defined and the autoscaler will not take any action for that metric.

### Increase load

- Now, we will see how the autoscaler reacts to increased load. We will start a container, and send an infinite loop of queries to the php-apache service (please run it in a different terminal): 

- First look at the services.

```bash
kubectl get svc
```

```bash
kubectl run -it --rm load-generator --image=busybox /bin/sh  

Hit enter for command prompt

while true; do wget -q -O- http://<puplic ip>:<port number of php-apache-service>; done 
```

Within a minute or so, we should see the higher CPU load by executing:

- Open new terminal and check the hpa.

```bash
$ kubectl get hpa 
```

On the watch board:
```bash
watch -n3 kubectl get service,hpa,pod -o wide
```


- Now, let's introduce load for to-do web app with load-generator pod as follows (in a couple of terminals):

```bash
$ kubectl exec -it load-generator -- /bin/sh
/ # while true; do wget -q -O- http://<puplic ip>:<port number of web-service> > /dev/null; done
```

Watch table
```bash
$ watch -n3 kubectl get service,hpa,pod -o wide

Every 3.0s: kubectl get service,hpa,pod -o wide                                                                     master: Thu Sep 17 11:29:19 2020
```

### Stop load

- We will finish our example by stopping the user load.

- In the terminal where we created the container with busybox image, terminate the load generation by typing `Ctrl` + `C`. Close the load introducing terminals grafecully and observe the behaviour at the watch board.

- Then we will verify the result state (after a minute or so):
  
```bash
$ kubectl get hpa 
```
  
```bash
$ kubectl get deployment
```

# References: 
https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

https://www.digitalocean.com/community/tutorials/webinar-series-deploying-and-scaling-microservices-in-kubernetes