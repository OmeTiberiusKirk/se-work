# RKE2

## Static ipv4 สำหรับการทำ lab

แก้ไข /etc/netplan/<แตกต่างกันไป>.yaml โดยประมาณ

```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:                  # เปลี่ยนเป็นชื่อการ์ดจอของคุณจากข้อ 1
      dhcp4: false          # ปิดการรับ IP อัตโนมัติ
      addresses:
        - 192.168.1.50/24   # ใส่ IP ที่คุณต้องการ Fix และ /24 (Subnet Mask)
      routes:
        - to: default
          via: 192.168.1.1  # ใส่ IP ของเร้าเตอร์คุณ (Gateway)
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1] # ใส่ DNS Server (ในภาพคือของ Google และ Cloudflare)
```

## Server Node Installation

```
curl -sfL https://get.rke2.io | sudo sh - && \
sudo mkdir -p /etc/rancher/rke2 && \
sudo mkdir -p /var/lib/rancher/rke2/server/manifests
```

## กำหนดค่า dedicated control plane (ไม่ให้มีการทำงานของ user workloads ใน control plane)

```
sudo tee /etc/rancher/rke2/config.yaml > /dev/null << EOF
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
EOF
```

## Selecting the CNI plugin.

```
sudo tee -a /etc/rancher/rke2/config.yaml > /dev/null << EOF
cni: cilium
disable-kube-proxy: true
EOF

sudo tee /var/lib/rancher/rke2/server/manifests/rke2-cilium-config.yaml > /dev/null << EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-cilium
  namespace: kube-system
spec:
  valuesContent: |-
    kubeProxyReplacement: true
    k8sServiceHost: "localhost"
    k8sServicePort: "6443"
    hubble:
      enabled: true
EOF
```

## Ingress NGINX to Traefik Migration

```
sudo tee -a /etc/rancher/rke2/config.yaml > /dev/null << EOF
ingress-controller:
  - traefik
EOF

sudo tee /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml > /dev/null << EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-traefik
  namespace: kube-system
spec:
  valuesContent: |-
    providers:
      kubernetesIngressNginx:
        enabled: true
        ingressClass: "rke2-ingress-nginx-migration"
        controllerClass: "rke2.cattle.io/ingress-nginx-migration"
EOF
```

## Starting a cluster

```
sudo systemctl enable rke2-server.service && \
sudo systemctl start rke2-server.service
```

## Post-installation

```
echo 'PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.profile && \
source ~/.profile && \
mkdir ~/.kube && \
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config && \
sudo chown $USER:$USER ~/.kube/config

kubectl scale deployment cilium-operator -n kube-system --replicas=1
```

## Agent (Worker) Node Installation

```
# get token from master
master@master:~$ sudo cat /var/lib/rancher/rke2/server/node-token

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sudo sh - && \
sudo systemctl enable rke2-agent.service && \
sudo mkdir -p /etc/rancher/rke2

sudo tee /etc/rancher/rke2/config.yaml > /dev/null << EOF
server: https://<nginx ip (load balance)>:9345
token: <node-token>
EOF

sudo systemctl start rke2-agent.service
```

```
kubectl create secret docker-registry ghcr-secret \
  --docker-server=https://ghcr.io \
  --docker-username=OmeTiberiusKirk \
  --docker-password=<GitHub Personal Access Token>
```
