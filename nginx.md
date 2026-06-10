# Nginx

## Configuring nginx

```
stream {
  # 1. Kubernetes API Server Port
  upstream rke2_api {
    server <master ip>:6443 max_fails=3 fail_timeout=30s;
    # ถ้าในอนาคตมี Master ตัวที่ 2, 3 ให้มาเพิ่มบรรทัดตรงนี้ได้เลย (HA)
  }
  server {
    listen 6443;
    proxy_pass rke2_api;
  }

  # 2. RKE2 Registration Port (สำหรับให้ Worker มารีจิสเตอร์)
  upstream rke2_register {
    server <master ip>:9345 max_fails=3 fail_timeout=30s;
  }
  server {
    listen 9345;
    proxy_pass rke2_register;
  }
}
```

## กำหนดค่าที่ server node

```
# /etc/rancher/rke2/config.yaml

tls-san:
  - "<load balance ip>"
```

