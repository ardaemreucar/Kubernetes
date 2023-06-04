# Hands-on Kubernetes-05 : Managing Secrets and ConfigMaps

Purpose of the this hands-on training is to give students the knowledge of Kubernetes Secrets and config-map


# Step 1 - Create Secrets

- The secret is defined using yaml. Below we'll be using the variables defined above and providing them with friendly labels which our application can use. This will create a collection of key/value secrets that can be accessed via the name, in this case secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  db_server: db.example.com
  db_username: admin
  db_password: P@ssw0rd!
```
- This yaml file can be used to with Kubectl to create our secret. When launching pods that require access to the secret we'll refer to the collection via the friendly-name.

- Use kubectl to create our secret.

```bash 
kubectl create -f secret.yaml
```

- The following command allows you to view all the secret collections defined.

```bash
kubectl get secrets
```
- The following command allows you to describe all the secret content defined.

```bash
kubectl describe secret mysecret
```

- In the next step we'll use these secrets via a Pod.

- create secret with cli imperative commands 
  
```bash
kubectl create secret generic mysecret2 --from-literal=db_server=db.example.com --from-literal=db_username=admin --from-literal=db_password=P@ssw0rd!
```
- Get the secrets 
  
```bash
kubectl get secrets
```
- show the history commands and not a secure way, create text files  includes these key-value data

```
echo  'db.example.com' > server.txt
echo 'admin' > username.txt
echo 'P@ssw0rd!'  > password.txt
```

```bash
kubectl create secret generic mysecret3 --from-file=db_server=server.txt --from-file=db_username=username.txt --from-file=db_password=password.txt
```
- Get the secrets 
  
```bash
kubectl get secrets
```
- create new secret4 by using ``secretconfig.json`` file
- 
```json
{
    "apiKey": "6bba108d4b2212f2c30c71dfa279e1f77cc5c3b2",
 }
```
- create secret4

```bash
kubectl create secret generic mysecret4 --from-file=secretconfig.json
```

- In the file **secret-env-pod.yaml** we've defined a Pod which has environment variables populated from the previously created secret.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secretpodvolume
spec:
  containers:
  - name: secretcontainer
    image: eagle79/secretimg
    volumeMounts:
    - name: secret-vol
      mountPath: /secret
  volumes:
  - name: secret-vol
    secret:
      secretName: mysecret
---
apiVersion: v1
kind: Pod
metadata:
  name: secretpodenv
spec:
  containers:
  - name: secretcontainer
    image: eagle79/secretimg
    env:
      - name: username
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: db_username
      - name: password
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: db_password
      - name: server
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: db_server
---
apiVersion: v1
kind: Pod
metadata:
  name: secretpodenvall
spec:
  containers:
  - name: secretcontainer
    image: eagle79/secretimg
    envFrom:
    - secretRef:
        name: mysecret
```

- To populate the environment variable we define the name, in this case SECRET_USERNAME, along with the name of the secrets collection and the key which containers the data.

- Launch the Pod using 
```
kubectl apply -f secret-env-pod.yaml
```
Once the Pods started, you output the populated environment variables.
- first pod
```
kubectl get pods
kubectl exec -it secretpodvolume -- bash
cd /
ls
cd secret
ls
```
- second pod
- 
```bash
kubectl exec -it secretpodenv -- printenv
```
- third pod
```bash
kubectl exec -it secretpodenvall -- printenv
```
- deleting secret objects
```bash
kubectl delete secret mysecret
```

# Step 2 - Create ConfigMap

- This YAML creates a ConfigMap with the value database set to mongodb, and database_uri, and keys set to the values in the YAML example code. Then, create the ConfigMap in the cluster using ``kubectl apply -f config-map.yaml``

```yaml
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: example-configmap 
data:
  # Configuration values can be set as key-value properties
  database: mongodb
  database_uri: mongodb://localhost:27017
  
  # Or set as complete file contents (even JSON!)
  keys: | 
    image.public.key=771 
    rsa.public.key=42
```
```bash
kubectl apply -f config-map.yaml
```

- The key to adding your ConfigMap as environment variables to your pods is the envFrom property in your Pod’s YAML. Set envFrom to a reference to the ConfigMap you’ve created.

```yaml
kind: Pod 
apiVersion: v1 
metadata:
  name: pod-env-var 
spec:
  containers:
    - name: env-var-configmap
      image: nginx
      envFrom:
        - configMapRef:
            name: example-configmap
```

```bash
kubectl apply -f pod-env-var.yaml
```

```bash
kubectl get po
kubectl exec -it pod-env-var -- env
```

- Imperative Commands

```
kubectl create configmap "configmap_ismi" --from-literal="key"="value" --from-file="key"="value_file" --from-file="value_file"

Ex: kubectl create configmap myconfigmap--from-literal=db_server=db.example.com --from-file=db_server=server.txt --from-file=config.json
```
- List ConfigMap objects

```
kubectl get configmap
```

- delete configmaps

```
kubectl delete configmap "configmap_ismi"

Ex: kubectl delete configmap my-configmap
```

echo -n '1f2d1e2e67df' | base64

echo 'MWYyZDFlMmU2N2Rm' | base64 --decode