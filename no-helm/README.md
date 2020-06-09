# To create

```
cd hello-kubernetes
kubectl create -f deployment
kubectl expose deployment hello-kubernetes --type=NodePort
minikube service hello-kubernetes --url
w3m $(minikube service hello-kubernetes --url)
curl$(minikube service hello-kubernetes --url)
```
