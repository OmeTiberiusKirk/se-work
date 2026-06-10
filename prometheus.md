# Prometheus

## Installing Prometheus

### ขั้นตอนที่ 1: สร้าง User และ Directory สำหรับ Prometheus

```
# สร้าง System User ชื่อ prometheus โดยไม่มีสิทธิ์ login เข้าหน้าจอ
sudo useradd --system --no-create-home --shell /bin/false prometheus

# สร้างโฟลเดอร์สำหรับเก็บไฟล์คอนฟิกและข้อมูล Metric
sudo mkdir -p /etc/prometheus && \
sudo mkdir -p /var/lib/prometheus
```

### ขั้นตอนที่ 2: ดาวน์โหลดและติดตั้ง Prometheus Binary

```
# ย้ายไปที่โฟลเดอร์ชั่วคราว
cd /tmp

# ดาวน์โหลด Prometheus v3.12.0 
wget https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-amd64.tar.gz

# แตกไฟล์ tar.gz
tar -xvf prometheus-3.12.0.linux-amd64.tar.gz

# ย้ายไฟล์ตัวรัน (Binary) ไปไว้ที่ /usr/local/bin
cd prometheus-3.12.0.linux-amd64 && \
sudo mv prometheus promtool /usr/local/bin/

# ย้ายไฟล์คอนฟิกตัวอย่าง ไปไว้ที่ /etc/prometheus
sudo mv prometheus.yml /etc/prometheus/
```

### ขั้นตอนที่ 3: กำหนดสิทธิ์การเข้าถึงไฟล์ (Ownership)

```
sudo chown -R prometheus:prometheus /etc/prometheus && \
sudo chown -R prometheus:prometheus /var/lib/prometheus && \
sudo chown prometheus:prometheus /usr/local/bin/prometheus && \
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

### ขั้นตอนที่ 4: สร้างไฟล์ Systemd Service

```
sudo vim /etc/systemd/system/prometheus.service
```

คัดลอกโค้ดด้านล่างนี้ไปวางในไฟล์:
```
[Unit]
Description=Prometheus Monitoring System
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.enable-lifecycle

Restart=always

[Install]
WantedBy=multi-user.target
```

### ขั้นตอนที่ 5: สั่งเริ่มต้นระบบและเปิดใช้งาน (Start Service)

```
# รีโหลดระบบ systemd เพื่อให้รู้จักไฟล์ที่เราเพิ่งสร้าง
sudo systemctl daemon-reload

# สั่งให้ Prometheus ทำงานทันที
sudo systemctl start prometheus

# สั่งให้ Prometheus ทำงานอัตโนมัติทุกครั้งที่เปิดเครื่อง
sudo systemctl enable prometheus
```

## Configuring Prometheus

### 1. ฝั่ง RKE2 Cluster: เปิดช่องทางให้ภายนอกเข้าถึง

แก้ไฟล์คอนฟิก RKE2 (/etc/rancher/rke2/config.yaml) ในทุกๆ Master (Server) Node:
```
# /etc/rancher/rke2/config.yaml
# ผูก Component ต่างๆ ให้ IP วงภายในมองเห็น (หรือใส่เป็น 0.0.0.0)
etcd-expose-metrics: true
kube-controller-manager-arg:
  - "bind-address=0.0.0.0"
kube-scheduler-arg:
  - "bind-address=0.0.0.0"
kube-proxy-arg:
  - "metrics-bind-address=0.0.0.0"
```

### 2. การจัดการสิทธิ์และการเข้าถึง (Authentication & Authorization)

รันคำสั่งด้านล่างนี้ใน RKE2 cluster เพื่อสร้าง Token:

```
# prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-prometheus
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-prometheus-binding
subjects:
- kind: ServiceAccount
  name: external-prometheus
  namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin # หรือใช้บทบาทระบบที่มีสิทธิ์เฉพาะในการดึง /metrics (เช่น system:auth-delegator) เพื่อความปลอดภัยสูงสุด
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Secret
metadata:
  name: external-prometheus-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: external-prometheus
type: kubernetes.io/service-account-token
```

ทำการดึง Token ออกมา เพื่อนำไปใส่ในเครื่อง Prometheus External:

```
kubectl get secret external-prometheus-token \
-n default -o jsonpath={.data.token} \
| base64 -d > ~/rke2-token.txt
```

### 3. ฝั่ง Prometheus (Native): คอนฟิกไฟล์ prometheus.yml

โครงสร้างมาตรฐานไฟล์คอนฟิกของ Prometheus ที่อยู่นอก Cluster เพื่อเข้ามาดึงข้อมูลในแต่ละส่วน

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # 1. ดึงข้อมูลจาก Kubernetes API Server (ใช้ดึงพวกข้อมูลภาพรวมของคลัสเตอร์)
  - job_name: 'rke2-apiserver'
    scheme: https
    tls_config:
      insecure_skip_verify: true # หากไม่ต้องการเช็ค SSL Cert ของ k8s (หากซีเรียสให้นำ CA ของ RKE2 มาใส่)
    bearer_token_file: '/etc/prometheus/rke2-token.txt'
    static_configs:
      - targets: ['192.168.122.2:6443'] # ใส่ IP ของ Master Nodes ของคุณ

  # 2. ดึงข้อมูลพฤติกรรมและการทำงานของ Kubelet (CPU/Memory ของ Pods ต่างๆ)
  - job_name: 'rke2-kubelet'
    scheme: https
    tls_config:
      insecure_skip_verify: true
    bearer_token_file: '/etc/prometheus/rke2-token.txt'
    static_configs:
      - targets: 
          - '192.168.122.2:10250' # ใส่ให้ครบทุก Node (ทั้ง Master และ Worker)

  # 3. ดึงข้อมูล CoreDNS (พอร์ตมาตรฐานของสถิติ DNS ภายในคลัสเตอร์)
  # หมายเหตุ: โดยปกติ CoreDNS จะเป็น Pod รันข้างใน หากจะคุยจากนอกวง แนะนำให้คุยผ่าน Service NodePort หรือดึงตรงหาก Network ทะลุกันได้
  # แต่ถ้ามองหา Control Plane ตัวอื่น เช่น etcd:
  - job_name: 'rke2-etcd'
    scheme: http # etcd-expose-metrics ของ RKE2 มักเป็น http บนพอร์ต 2381
    static_configs:
      - targets: ['192.168.122.2:2381']

  # 4. ดึงข้อมูล Kube-Scheduler และ Controller-Manager
  - job_name: 'rke2-scheduler'
    scheme: https
    tls_config:
      insecure_skip_verify: true
    bearer_token_file: '/etc/prometheus/rke2-token.txt'
    static_configs:
      - targets: ['192.168.122.2:10259']

  - job_name: 'rke2-controller-manager'
    scheme: https
    tls_config:
      insecure_skip_verify: true
    bearer_token_file: '/etc/prometheus/rke2-token.txt'
    static_configs:
      - targets: ['192.168.122.2:10257']
```

 Please provide instructions on how to install Kube-State-Metrics using daemonset.