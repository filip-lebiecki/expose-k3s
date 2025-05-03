# Install k3s in HA mode

```
node1 root: curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server --cluster-init --cluster-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/16 --disable=servicelb --disable-network-policy --disable=traefik
node2 root: curl -sfL https://get.k3s.io | K3S_URL=https://192.168.12.27:6443 K3S_TOKEN=SECRET sh -s - server --cluster-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/16 --disable=servicelb --disable-network-policy --disable=traefik
node3 root: curl -sfL https://get.k3s.io | K3S_URL=https://192.168.12.27:6443 K3S_TOKEN=SECRET sh -s - server --cluster-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/16 --disable=servicelb --disable-network-policy --disable=traefik

node1 as user:
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml .kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
kubectl describe node node1 node2 node3 | grep Taints
```

# Install Traefik as DaemonSet

```
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik --namespace kube-system --set service.type=NodePort --set deployment.kind=DaemonSet
curl -H "Host: whoami.linuxcloudhacks.ovh" 192.168.12.27:32538
```

# ExternalIP

```
Add second IP /etc/netplan/network.yaml
netplan try
edit NodePort Traefik Service, add
externalIPs:
- 192.168.12.30

curl -H "Host: whoami.linuxcloudhacks.ovh" http://192.168.12.30:32538
```

# Floating IP (KeepaliveD)

```
Remove secondary IP

node1:
sudo vi /etc/netplan/network.yaml      
- 192.168.12.30

sudo netplan try
ip -4 -br a s dev eth0
cat /etc/keepalived/keepalived.conf
sudo systemctl start keepalived 
sudo systemctl status keepalived
node2: cat /etc/keepalived/keepalived.conf     
node2: sudo systemctl start keepalived
node3: sudo systemctl start keepalived
```

# MetalLB Layer2

```
helm repo add metallb https://metallb.github.io/metallb 
helm repo update 
helm install metallb metallb/metallb --namespace kube-system
cat metallb-advertise.yaml
kubectl apply -f metallb-advertise.yaml
cat metallb-addr-pool.yaml
kubectl apply -f metallb-addr-pool.yaml
helm install traefik traefik/traefik --namespace kube-system --set service.type=LoadBalancer --set deployment.kind=DaemonSet
kubectl apply -f whoami.yaml
```

# Cloudflare Tunnels multipe replicas

```
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg 
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
apt update
apt install cloudflared
systemctl disable --now cloudflared
cloudflared tunnel login
cloudflared tunnel create my-tunnel
kubectl create secret generic tunnel-credentials --from-file=credentials.json=.cloudflared/44aea25e-ab1f-47fd-8d2a-258e58bd4a1e.json 
kubectl apply -f cloudflared-ds-ingress.yaml  
cloudflared tunnel route dns my-tunnel "*.linuxcloudhacks.ovh"
```
