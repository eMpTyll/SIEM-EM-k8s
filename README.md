# Wazuh, Elasticsearch, and Kibana Kubernetes Deployment

This repository contains Kubernetes manifests for deploying a complete monitoring and security stack consisting of:

- Wazuh Manager (security monitoring)
- Wazuh Indexer
- Elasticsearch (2-node cluster)
- Kibana (visualization dashboard)
- Logstash (processing and transforming logs)

## Tóm tắt kiến trúc
- Wazuh Manager: Thu thập và phân tích log bảo mật từ các agent
- Logstash: Xử lý logs từ Wazuh và gửi vào Elasticsearch
- Elasticsearch: Lưu trữ và đánh index dữ liệu
- Kibana: Giao diện trực quan hóa và phân tích dữ liệu
- Wazuh Indexer: Cung cấp khả năng đánh index bổ sung cho dữ liệu Wazuh

## Phiên bản và tương thích
- Elasticsearch/Kibana/Logstash: 8.12.2
- Wazuh và Wazuh Indexer: 4.7.3

**Lưu ý về tương thích:** Cấu hình đã được tối ưu hóa để sử dụng Wazuh 4.7.3 với Elasticsearch 8.12.2 để đảm bảo tính tương thích tốt nhất. Phiên bản Elasticsearch 8.x được khuyến nghị sử dụng với Wazuh 4.5.x.

## Tích hợp Wazuh với Elasticsearch thông qua Logstash

Hệ thống này sử dụng Logstash để chuyển tiếp dữ liệu từ file alerts.json của Wazuh đến Elasticsearch. Cấu hình này đã được tối ưu hóa dựa trên hướng dẫn chính thức của Wazuh:

### Các thành phần chính trong cấu hình

1. **Plugin Logstash-output-elasticsearch**:
   - Được cài đặt tự động thông qua init container
   - Cho phép Logstash ghi dữ liệu vào Elasticsearch

2. **Template Elasticsearch**:
   - File template `wazuh.json` được cung cấp để đảm bảo Elasticsearch đánh index dữ liệu chính xác
   - Cấu hình giới hạn trường đã được tăng lên 10000 (mặc định chỉ là 1000)
   - Cấu hình `refresh_interval` được đặt là 5 giây

3. **Pipeline Logstash**:
   - Đọc dữ liệu từ file `/var/ossec/logs/alerts/alerts.json`
   - Chuyển đổi và gửi dữ liệu đến Elasticsearch với định dạng index `wazuh-alerts-4.x-%{+YYYY.MM.dd}`
   - Sử dụng thông tin xác thực từ Kubernetes secrets

4. **Bảo mật**:
   - Hiện tại SSL được đặt thành `false` cho môi trường phát triển
   - Để triển khai trong môi trường production, nên bật SSL và thêm chứng chỉ CA

## Điều kiện tiên quyết

* Trước khi triển khai, đảm bảo bạn có:
* Kubernetes cluster running on VMware vSphere
* kubectl installed and configured to access your cluster
* Sufficient resources in your vSphere environment:

  * At least 6 vCPUs and 12GB RAM available
  * At least 100GB of storage capacity
* Ingress controller installed in your cluster (e.g., NGINX Ingress)
* Domain names configured for accessing Kibana and Wazuh API

## Deployment Instructions

### 1. Clone this repository

```
git clone https://github.com/yourusername/wazuh-elk-k8s.git
cd wazuh-elk-k8s
```

### 2. Update configuration files

Before deploying, update the following:

- In `elasticsearch-secret.yaml` and `wazuh-secret.yaml`:
- Change the default passwords (base64 encode your passwords)

```shellscript
echo -n "mật-khẩu-an-toàn-của-bạn" | base64
```

- In `ingress.yaml`:
- Update the hostnames to match your domain names

### 3. Create the monitoring namespace

```shellscript
kubectl apply -f namespace.yaml
```

### 4. Create the vSphere storage class

```shellscript
kubectl apply -f vsphere-storage-class.yaml
```

### 5. Create secrets

```shellscript
kubectl apply -f elasticsearch-secret.yaml
kubectl apply -f elasticsearch-certs.yaml
kubectl apply -f wazuh-secret.yaml
```

### 6. Deploy Elasticsearch

```shellscript
kubectl apply -f elasticsearch-statefulset.yaml
```

Wait for the Elasticsearch pods to be in Running state:

```shellscript
kubectl get pods -n monitoring -l app=elasticsearch -w
```

### 7. Deploy Wazuh components

First, create the persistent volume claims:

```shellscript
kubectl apply -f wazuh-manager-pvcs.yaml
```

Then deploy the Wazuh indexer:

```shellscript
kubectl apply -f wazuh-indexer-statefulset.yaml
```

Wait for the Wazuh indexer pod to be in Running state:

```shellscript
kubectl get pods -n monitoring -l app=wazuh-indexer -w
```

Next, create the ConfigMap for Logstash:

```shellscript
kubectl apply -f logstash-config-map.yaml
```

Finally, deploy the Wazuh manager with Logstash:

```shellscript
kubectl apply -f wazuh-manager-logstash-deployment.yaml
```

### 8. Deploy Kibana

```shellscript
kubectl apply -f kibana-deployment.yaml
```

Wait for the Kibana pod to be in Running state:

```shellscript
kubectl get pods -n monitoring -l app=kibana -w
```

### 9. Create the ingress for external access

```shellscript
kubectl apply -f ingress.yaml
```

### 10. Change hosts file

1. Get IP of Ingress Controller

```shellscript
kubectl get svc -n ingress-nginx
```

Find ingress-nginx-controller and note value of EXTERNAL-IP. If dont have EXTERNAL-IP (In case of local deployment), use node IP:

```shellscript
kubectl get nodes -o wide
```

Note the node IP (Ex: 192.168.1.100)

2. Config the hosts file

```shellscript
sudo nano /etc/hosts
```

Add the following line in the end of file:

```shellscript
192.168.1.100 kibanacypeace.com
```

## Post-Deployment Configuration

### Verify all components are running

```shellscript
kubectl get pods -n monitoring
```
All pods should be in the `Running` state.

### Verify Logstash is processing Wazuh alerts

Kiểm tra logs của container Logstash để đảm bảo nó đang xử lý dữ liệu từ Wazuh:

```shellscript
kubectl logs -n monitoring -l app=wazuh-manager -c logstash
```

Bạn nên thấy các thông báo về việc Logstash đang xử lý sự kiện từ file alerts.json và gửi đến Elasticsearch.

### Access Kibana

1. Navigate to `https://kibana.yourdomain.com` in your browser
2. Log in with:
3. Username: `elastic`
4. Password: (the password you set in `elasticsearch-secret.yaml`)

### Configure Wazuh in Kibana

1. In Kibana, navigate to the Wazuh app (you may need to install it from the Kibana plugin menu)
2. Configure the connection to the Wazuh manager:
3. URL: `http://wazuh-manager.monitoring.svc.cluster.local:55000` or `http://wazuh.yourdomain.com`
4. Port: `55000`
5. Username: `wazuh`
6. Password: (the password you set in `wazuh-secret.yaml`)

### Add Agents to Wazuh

1. In the Wazuh app in Kibana, go to "Agents" and click "Deploy new agent"
2. Follow the instructions to deploy agents on your systems

## Lưu ý về cấu hình

### 1. Storage Class

Tất cả các thành phần trong hệ thống đã được cấu hình để sử dụng `vsphere-storage` storage class, phù hợp với môi trường VMware vSphere. Storage class này được cấu hình trong file `vsphere-storage-class.yaml`.

### 2. Bảo mật

**Cấu hình hiện tại:** Hệ thống được cấu hình để chạy không có SSL và có `xpack.security.enabled: "false"` trong Elasticsearch.

**Khuyến nghị cho môi trường sản xuất:**
- Bật SSL và security features
- Cấu hình TLS giữa các thành phần
- Sử dụng mật khẩu mạnh và thay đổi định kỳ

### 3. Điều chỉnh Logstash

Cấu hình Logstash có thể được điều chỉnh bằng cách sửa đổi file `logstash-config-map.yaml`. Một số điều chỉnh quan trọng:

- **Bật SSL**: Trong môi trường sản xuất, bỏ comment các dòng liên quan đến SSL và cung cấp chứng chỉ CA.
- **Tùy chỉnh Template**: Có thể tăng giới hạn trường (hiện tại là 10000) nếu cần thiết.
- **Tuning Performance**: Điều chỉnh `refresh_interval` và các tham số khác để tối ưu hóa hiệu suất.

## Kiểm tra và khắc phục sự cố

### Kiểm tra logs

```shellscript
# Logs của Elasticsearch
kubectl logs -n monitoring -l app=elasticsearch

# Logs của Wazuh Manager
kubectl logs -n monitoring -l app=wazuh-manager -c wazuh-manager

# Logs của Logstash
kubectl logs -n monitoring -l app=wazuh-manager -c logstash

# Logs của Kibana
kubectl logs -n monitoring -l app=kibana
```

### Kiểm tra kết nối giữa các thành phần

```shellscript
# Truy cập vào pod Wazuh Manager
kubectl exec -it -n monitoring $(kubectl get pods -n monitoring -l app=wazuh-manager -o name | head -n 1) -c wazuh-manager -- /bin/bash

# Kiểm tra kết nối đến Elasticsearch
curl -v http://elasticsearch-api:9200

# Kiểm tra trạng thái Wazuh
/var/ossec/bin/ossec-control status

# Kiểm tra file alerts.json
ls -la /var/ossec/logs/alerts/
cat /var/ossec/logs/alerts/alerts.json | tail -n 20
```

### Khắc phục sự cố Logstash

Nếu Logstash không gửi dữ liệu đến Elasticsearch, hãy kiểm tra:

1. Plugin logstash-output-elasticsearch đã được cài đặt:
```shellscript
kubectl exec -it -n monitoring $(kubectl get pods -n monitoring -l app=wazuh-manager -o name | head -n 1) -c logstash -- /usr/share/logstash/bin/logstash-plugin list | grep elasticsearch
```

2. Logstash có quyền đọc file alerts.json:
```shellscript
kubectl exec -it -n monitoring $(kubectl get pods -n monitoring -l app=wazuh-manager -o name | head -n 1) -c logstash -- ls -la /var/ossec/logs/alerts/
```

3. Cấu hình Logstash có lỗi nào không:
```shellscript
kubectl exec -it -n monitoring $(kubectl get pods -n monitoring -l app=wazuh-manager -o name | head -n 1) -c logstash -- cat /etc/logstash/conf.d/wazuh-elasticsearch.conf
```

## Backup và khôi phục

### Tạo snapshot Elasticsearch

1. Đăng ký một repository snapshot trong Elasticsearch:

```shellscript
curl -X PUT "http://elasticsearch-api:9200/_snapshot/backup_repo" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/usr/share/elasticsearch/data/backups"
  }
}'
```

2. Tạo snapshot:

```shellscript
curl -X PUT "http://elasticsearch-api:9200/_snapshot/backup_repo/snapshot_1?wait_for_completion=true"
```

### Backup cấu hình Wazuh

```shellscript
kubectl exec -it -n monitoring $(kubectl get pods -n monitoring -l app=wazuh-manager -o name | head -n 1) -c wazuh-manager -- tar -czf /tmp/wazuh-etc.tar.gz /var/ossec/etc

kubectl cp monitoring/$(kubectl get pods -n monitoring -l app=wazuh-manager -o name | head -n 1 | sed 's|pod/||'):tmp/wazuh-etc.tar.gz ./wazuh-etc-backup.tar.gz -c wazuh-manager
```

## Nâng cấp

Quy trình nâng cấp chi tiết sẽ phụ thuộc vào phiên bản hiện tại và phiên bản đích. Nhìn chung, quy trình sẽ bao gồm:

1. Backup tất cả dữ liệu và cấu hình
2. Cập nhật các image tags trong file deployment
3. Áp dụng các thay đổi theo thứ tự:
   - Elasticsearch
   - Wazuh Indexer
   - Wazuh Manager và Logstash
   - Kibana
4. Kiểm tra logs để đảm bảo mọi thứ hoạt động bình thường
5. Kiểm tra tính năng và hoạt động của hệ thống

## Kết luận

Hệ thống Wazuh, Elasticsearch, và Kibana được cấu hình trong repository này cung cấp một giải pháp mạnh mẽ cho giám sát bảo mật. Các điểm quan trọng:

- Đã tối ưu hóa tương thích giữa Wazuh 4.7.3 và Elasticsearch 8.12.2
- Đã thống nhất sử dụng vsphere-storage cho tất cả các thành phần
- Cung cấp quy trình triển khai chi tiết và hướng dẫn cấu hình
- Tích hợp Logstash để chuyển tiếp dữ liệu từ Wazuh đến Elasticsearch theo khuyến nghị chính thức

Sau khi triển khai, hãy kiểm tra thường xuyên logs và hiệu suất hệ thống để đảm bảo mọi thứ hoạt động tốt.
