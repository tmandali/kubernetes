# Install Kubernetes on CNI

## Calico CNI Install
Calico operatörünü dağıtmak için ana düğümde aşağıdaki komutları çalıştırın.

https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

``` bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.10.0.0\/16/g' custom-resources.yaml

kubectl create -f custom-resources.yaml

watch kubectl get pods -n calico-system
kubectl get nodes -o wide

# Calicoctl Install 
curl -L https://github.com/projectcalico/calico/releases/download/v3.28.0/calicoctl-linux-arm64 -o calicoctl

chmod +x ./calicoctl
```
**Test**
``` bash
kubectl create deployment nginx --image=nginx
kubectl create service nodeport nginx --tcp 80:80
nc -zv 10.96.0.1 80
#kubectl patch service nginx -p '{"spec":{"externalTrafficPolicy":"Local"}}'
#ip r
#kubectl scale --replicas=2 deployment/nginx
#ip r
```