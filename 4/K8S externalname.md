# Create an ExternalName service called externalname that points to google.com


```bash
$ kubectl create service externalname externalname --external-name=google.com
```

Describe the externalname Service. Note that it does NOT have an internal IP or other normal service attributes.

```bash
$ kubectl describe service externalname
```

Lastly, look at the generated DNS record has been created for the Service by using nslookup within the example-pod Pod. It should return the IP of google.com.

```bash
$ kubectl exec pod-example -- nslookup externalname.default.svc.cluster.local
```


-----

- Useful link to practice K8s

https://killercoda.com/playgrounds/scenario/kubernetes

