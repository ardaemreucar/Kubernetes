# Kubernetes Networking Quick Lab

kubectl create deployment my-deploy --image nginx --replicas 3 --dry-run client -o yaml

kubectl create deployment my-deploy --image nginx --replicas 3



kubectl expose deployment my-deploy --port 80 --dry-run client -o yaml

kubectl expose deployment my-deploy --port 80


kubectl get svc

kubectl run kutu --image busybox -- command sleep 3600

kubectl exec -it kutu -- sh

/# wget svcIP or svcname

this is the way communication inside cluster takes place


traffic from outside cluster

kubectl edit svc my-deploy

change type to NodePort

kubectl get svc

kubectl get nodes -o wide

node IP is private. we need to get into the node. node is a docker container. we need to get into the container.

docker exec -it ..

curl localhost:nodePort port

