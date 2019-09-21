# kubernetes-cluster

```
iptables -I INPUT -j ACCEPT

sudo swapoff -a 
sudo sed -i '/ swap / s/^/#/' /etc/fstab

reboot
```

```
apt-get install -y kubelet kubeadm kubectl

cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl restart docker

cat > /etc/default/kubelet <<EOF
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
EOF
```

```
kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=192.168.8.106
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/canal.yaml
```

```
# need for work ingress 
snap install socat
```

```
# after reboot 
sudo systemctl restart docker
sudo systemctl restart kubelet
```

```
# listeners ports 
socat -d TCP4-LISTEN:80,fork TCP4:127.0.0.1:30380 </dev/null &
socat -d TCP4-LISTEN:443,fork TCP4:127.0.0.1:30443 </dev/null &
```

```
# inastall apps 

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
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
kubectl patch svc ingress-nginx -n ingress-nginx -p '{"spec": {"ports": [{"name": "http","port": 80,"protocol": "TCP","targetPort": 80, "nodePort": 30380}, {"name": "https","port": 443,"protocol": "TCP","targetPort": 443, "nodePort": 30443}]}}'

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