# Loadbalancer

## MetalLB

On-prem kubernetes clusterların External IP almaları için kurulur 

**Install**
https://metallb.universe.tf/installation/#installation-by-manifest

``` bash
# Preparation
kubectl edit configmap -n kube-system kube-proxy

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system

# To install MetalLB, apply the manifest:
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

# IMPORTANT: Make sure you exclude this slice from the address pool of your DHCP server, otherwise you will run into troubles.
 
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF

kubectl wait pod --all --for=condition=Ready --namespace=metallb-system --timeout=300s
kubectl get all --namespace metallb-system
```

**Test**
``` bash
# Let's create a simple web server and the associated service:
kubectl create deployment demo --image=httpd --port=80 
kubectl expose deployment demo --type LoadBalancer --port 80 --target-port 8080
kubectl scale --replicas=3 deployment demo

# Change external ip:
for i in {1..5}; do curl http://192.168.1.240; done
```