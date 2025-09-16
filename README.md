# Grafana: monitor Nginx Proxy Manager website

![banner](https://github.com/williamdonze/nginx-proxy-manager-grafana/assets/146176936/87d26b9d-9275-49e2-818a-2a3bb93d5441)
---
This project is part of the blog article [Grafana: monitor Nginx Proxy Manager website](https://medium.com/@williamdonze/grafana-monitor-nginx-proxy-manager-website-4d03b60c761f)

## Code
- [(website-all.json) - JSON all website dashboard](https://github.com/williamdonze/nginx-proxy-manager-grafana/blob/main/website-all.json)
- [(websites.json) - JSON single website dashboard](https://github.com/williamdonze/nginx-proxy-manager-grafana/blob/main/websites.json)

## 🤝 Support

Contributions, issues, and feature requests are welcome!

Give a ⭐️ if you like this project!

Oke 👍 aku bisa bantu ubah konten yang kamu tempel jadi file markdown (`.md`) dengan struktur rapi. Berikut hasil konversinya:

````markdown
# Monitor Nginx Proxy Manager with Grafana

## Introduction
To monitor the Nginx logs of your websites, you’ll need 4 different tools:  

- **Nginx Proxy Manager** (or simple Nginx but you’ll have to adapt this tutorial then)  
- **Promtail**  
- **Loki**  
- **Grafana**  

In this guide, we’ll use the **Docker version** of each of these tools.

---

## Dashboards
With this tutorial you should be able to have Grafana dashboards like these:

- **Websites in general**
- **Specific website (with variable filter)**

---

## Step 1 — Configure a custom `log_format` on Nginx Proxy Manager

### Example `docker-compose.yml` for NPM

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager    
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
````

### Create custom log format

```bash
cd /data/compose/nginx # change to the correct directory
mkdir custom
```

Create `http_top.conf`:

```nginx
log_format json_analytics escape=json '{'
       '"time_local": "$time_local", '
       '"remote_addr": "$remote_addr", '
       '"request_uri": "$request_uri", '
       '"status": "$status", '
       '"server_name": "$server_name", '
       '"request_time": "$request_time", '
       '"request_method": "$request_method", '
       '"bytes_sent": "$bytes_sent", '
       '"http_host": "$http_host", '
       '"http_x_forwarded_for": "$http_x_forwarded_for", '
       '"http_cookie": "$http_cookie", '
       '"server_protocol": "$server_protocol", '
       '"upstream_addr": "$upstream_addr", '
       '"upstream_response_time": "$upstream_response_time", '
       '"ssl_protocol": "$ssl_protocol", '
       '"ssl_cipher": "$ssl_cipher", '
       '"http_user_agent": "$http_user_agent", '
       '"remote_user": "$remote_user" '
   '}';
```

Create `server_proxy.conf`:

```nginx
access_log /data/logs/all_proxy_access.log json_analytics;
error_log /data/logs/all_proxy_error.log warn;
```

Restart Nginx container and you should see structured JSON logs like:

```json
{
  "time_local": "19/Sep/2023:15:04:54 +0000", 
  "remote_addr": "192.168.10.77", 
  "request_uri": "/webpage", 
  "status": "200", 
  "server_name": "your-domain.com", 
  "request_time": "0.002", 
  "request_method": "GET", 
  "bytes_sent": "356", 
  "http_host": "your-domain.com", 
  "upstream_addr": "192.168.1.13:8080", 
  "upstream_response_time": "0.003", 
  "ssl_protocol": "TLSv1.3"
}
```

---

## Step 2 — Set up log scraping tools

We’ll run **Promtail**, **Loki**, and **Grafana** with Docker Compose.

### Example `docker-compose.yml`

```yaml
version: "3"

networks:
  loki:

services:
  loki:
    image: grafana/loki:2.8.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki
    volumes:
      - ./loki-data:/loki
      - ./config-loki:/etc/loki

  promtail:
    image: grafana/promtail:2.8.0
    volumes:
      - /data/compose/nginx/logs/:/var/log
      - ./config-promtail/config.yaml:/etc/promtail/config.yaml
    networks:
      - loki

  grafana:
    image: grafana/grafana:9.3.13
    ports:
      - "3000:3000"
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=false
    entrypoint:
      - sh
      - -euc
      - |
        mkdir -p /etc/grafana/provisioning/datasources
        cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
        apiVersion: 1
        datasources:
        - name: Loki
          type: loki
          access: proxy 
          orgId: 1
          url: http://loki:3100
          isDefault: true
        EOF
        /run.sh
    networks:
      - loki
    volumes:
      - ./grafana-data:/var/lib/grafana
```

### Folder structure

```
├── config-loki
│   └── local-config.yaml
├── config-promtail
│   └── config.yaml
├── docker-compose.yaml
├── grafana-data  # chown 472
└── loki-data     # chown 10001
```

### Promtail config — `config.yaml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
```

### Loki config — `local-config.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

query_scheduler:
  max_outstanding_requests_per_tenant: 2048
```

Run containers:

```bash
docker-compose up -d
```

---

## Step 3 — Setting up Grafana

1. Open [http://your-server-address:3000](http://your-server-address:3000)
2. Default login: `admin / admin`
3. Change password
4. Check **Loki** data source (already provisioned in step 2)
5. Import dashboards via JSON file

---

## Step 4 — Dashboards

* **website-all.json** → View all traffic across NPM
* **websites.json** → View traffic by site (with variables)

Edit **Variables → website** to replace default domains with your own.

---

## Step 5 — Conclusion

You’re now ready to monitor your Nginx logs! 🎉
This setup provides a structured log pipeline with **Nginx → Promtail → Loki → Grafana dashboards**.

If you want to dive deeper into Grafana and Loki, check out additional docs and tutorials.

```

---

Mau aku bikinkan **file `.md` langsung** (siap download), atau cukup kamu copy-paste hasil di atas?
```
