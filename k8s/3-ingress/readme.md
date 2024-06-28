# Install Kubernetes on Ingress

## Nginx Ingress Controller
**Install**

https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

https://github.com/kubernetes/ingress-nginx/

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

kubectl get pods --namespace=ingress-nginx
```
**Local Test**
``` bash
# Let's create a simple web server and the associated service:
kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo

# Then create an ingress resource. The following example uses a host that maps to localhost
kubectl create ingress demo-localhost --class=nginx \
  --rule="demo.localdev.me/*=demo:80"

kubectl get svc --namespace=ingress-nginx 

curl --resolve demo.localdev.me:192.168.1.241 http://demo.localdev.me

# Now, forward a local port to the ingress controller:
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80

curl --resolve demo.localdev.me:8080:127.0.0.1 http://demo.localdev.me:8080

```