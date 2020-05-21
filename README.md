# install k8s-cluster with cert-manager on Ubuntu 18


#### init node
```
iptables -I INPUT -j ACCEPT

sudo swapoff -a 
sudo sed -i '/ swap / s/^/#/' /etc/fstab

reboot
```

#### install required programs
```
apt-get install -y kubelet kubeadm kubectl

cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "insecure-registries" : ["localhost:32000"]
}
EOF
sudo systemctl restart docker

cat > /etc/default/kubelet <<EOF
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
EOF
```

```
# need for work ingress 
snap install socat
```


#### start kuberenetes node
```
kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=192.168.8.106
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/canal.yaml
```

```
# listeners ports 
socat -d TCP4-LISTEN:80,fork TCP4:127.0.0.1:30380 </dev/null &
socat -d TCP4-LISTEN:443,fork TCP4:127.0.0.1:30443 </dev/null &
```

#### inastall kubernetes apps 
```
cat > /opt/rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF
kubectl apply -f /opt/rbac.yaml

snap install helm --classic
helm init --service-account tiller --output yaml | sed 's@apiVersion: extensions/v1beta1@apiVersion: apps/v1@' | sed 's@  replicas: 1@  replicas: 1\n  selector: {"matchLabels": {"app": "helm", "name": "tiller"}}@' | kubectl apply -f -

# helm version 3 https://okteto.com/blog/early-look-at-helm-3/

#ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-0.32.0/deploy/static/provider/baremetal/deploy.yaml
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"ports": [{"name": "http","port": 80,"protocol": "TCP","targetPort": 80, "nodePort": 30380}, {"name": "https","port": 443,"protocol": "TCP","targetPort": 443, "nodePort": 30443}]}}'

# dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
#export KUBE_EDITOR="nano"
#kubectl edit svc/kubernetes-dashboard -n kubernetes-dashboard
kubectl patch svc kubernetes-dashboard -n kubernetes-dashboard -p '{"spec": {"ports": [{"port": 443,"protocol": "TCP","targetPort": 8443, "nodePort": 30345}], "type": "NodePort"}}'
kubectl -n kube-system describe secrets  $(kubectl -n kube-system get secret | grep tiller  | awk -F '[= ]+' '{ print $1 }')

# open in firefox https://k8s:30345/#/node?namespace=kube-system

# all permissions for
kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=admin --user=kubelet --group=system:serviceaccounts

```

#### docker registry 
```
docker run -d \
  --restart=always \
  --name registry \
  -v /mnt/kubernetes/storage/registry:/var/lib/registry \
  -p 32000:5000 \
  --name registry \
  registry:2
```

##### force docker stop registry
```
docker exec registry reboot
docker stop registry
```

##### fail delete container in docker (signaling init process caused "permission denied")
```
sudo aa-status
sudo systemctl disable apparmor.service --now
sudo service apparmor teardown
sudo aa-status
```

### after reboot scripts
```
sudo systemctl restart docker
sudo systemctl restart kubelet

docker run -d --rm -p 32000:5000 --name registry registry:2

# mount hdd disk 
mount  /dev/sdb1  /mnt/

socat -d TCP4-LISTEN:80,fork TCP4:127.0.0.1:30380 </dev/null &
socat -d TCP4-LISTEN:443,fork TCP4:127.0.0.1:30443 </dev/null &
```

### helm 3
##### switch context
```
kubectl config set-context kubernetes-admin@kubernetes --namespace xxxxx
helm list
```

### cert-manager
```
# https://docs.cert-manager.io/en/latest/getting-started/install/kubernetes.html
# Install the CustomResourceDefinition resources separately

kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.11/deploy/manifests/00-crds.yaml
kubectl create namespace cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml
```

#### add cert-manager issuer

```
cat > /opt/prod-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: bombascter@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
       - http01:
           ingress:
             class:  nginx
EOF
kubectl apply -f /opt/prod-issuer.yaml -n scraper
kubectl apply -f /opt/prod-issuer.yaml -n elasticsearch
kubectl apply -f /opt/prod-issuer.yaml -n jenkins
```

##### example ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myname
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.secretName }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            backend:
              serviceName: {{ template "docs.fullname" . }}
              servicePort: {{ .Values.service.port }}
```

```
kubectl get certificate -n scraper
```

##### pod errors 
```
kubectl get pods -n jenkins | grep Evicted | awk '{print $1}' | xargs kubectl delete pod  -n jenkins
kubectl get pods -n kubernetes-dashboard | grep Evicted | awk '{print $1}' | xargs kubectl delete pod  -n kubernetes-dashboard
kubectl get pods -n ingress-nginx | grep Evicted | awk '{print $1}' | xargs kubectl delete pod  -n ingress-nginx
```
### Dinamic storage class
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
# storageClassName: local-path
```
