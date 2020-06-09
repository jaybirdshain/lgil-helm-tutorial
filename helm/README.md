Worked through the no-helm deployment first

```
cd helm
helm create hello-kubernetes # creates the structure
kubectl get svc hello-kubernetes -o yaml > hello-kubernetes/templates/service.yaml
kubectl get deploy hello-kubernetes -o yaml > hello-kubernetes/templates/deploy.yaml
kubectl delete svc,deploy[,others] hello-kubernetes #of existing so it can be replaced with helm chart
helm install hello-kubernetes ./hello-kubernetes
w3m $(minikube service hello-kubernetes --url)
```
