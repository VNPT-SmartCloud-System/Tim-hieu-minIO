# Hướng dẫn tích hợp giám sát minIO cluster sử dụng Prometheus và Grafana
## Đầu bài:
Giám sát tải tới cụm minIO và trạng thái sử dụng tài nguyên của minIO cluster.

## Giải pháp:
 - Prometheus để thu thập metric của minIO cluster.
 - Grafana để visualize metric thành biểu đồ theo dõi.

## Mô hình lab:
 - minIO cluster: gồm 4 node, GW: 10.159.19.81
 - Prometheus + Grafana: IP 10.159.19.85


## 1. Cài đặt promethus để giám sát minIO cluster
### 1.1. Trên host minIO 1, sử dụng mc để cấu hình (mc là minIO client để tương tác với minIO cluster)
```sh
docker run --restart=always -it --entrypoint=/bin/sh minio/mc
mc config host add longlq http://10.159.19.81 access_key secret_key --api S3v4
```

 - *longlq*: alias đặt cho cluster, sử dụng cho các thao tác sau với mc.
 - *access_key*: access key của minIO cluster
 - *secret_key*: secret key của minIO cluster

### 1.2. Kiểm tra việc login vào minIO cluster bằng cách list thử các bucket và object trong cluster
```sh
mc ls longlq
```

Lệnh trên sẽ liệt kê các bucket và object đang tồn tai trong minIO cluster, kết quả:
```sh
[2020-03-06 02:10:21 UTC]      0B egov-bucket/
[2020-02-20 05:57:49 UTC]      0B test/
[2020-03-04 08:46:23 UTC]      0B test1/
[2020-03-04 08:46:23 UTC]      0B test2/
[2020-02-24 04:11:55 UTC]      0B test3/
[2020-02-24 04:12:29 UTC]      0B test4/
[2020-02-24 04:13:43 UTC]      0B test5/
[2020-03-21 18:00:36 UTC]      0B test6/
[2020-03-23 08:40:14 UTC]      0B test7/
[2020-02-24 04:12:35 UTC]      0B test8/
```

### 1.3. Sửa file docker-compose của minIO cluster (ở đây là **minio-stack.yml**) để cập nhật thêm cấu hình cho phép cluster có thể pull metric minIO từ máy khác local

minIO sử dụng 2 cơ chế xác thực cho Prometheus giao tiếp là *jwt* và *public*. Với việc dùng cơ chế *public* sẽ cho phép Prometheus truy cập vào minIO mà không cần xác thực.

Sửa file **minio-stack.yml**, trong thẻ **environment** bổ sung thêm: **MINIO_PROMETHEUS_AUTH_TYPE: public**

### 1.4. Update lại docker swarm stack để cập nhật cấu hình mới vào cluster
```sh
docker stack deploy -c minio-stack.yml minio-cluster
```
 Chú ý: 
 - *minio-cluster*: tên của minIO cluster, khi update tên cluster phải trùng với tên của cluster đang chạy.

### 1.5. Chạy lệnh trên minIO cluster để xuất metric cho prometheus
```sh
mc admin prometheus generate longlq
```
Kết quả:
```sh
scrape_configs:
- job_name: minio-job  
  bearer_token: eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJleHAiOjQ3Mzg0MTI1NTIsImlzcyI6InByb21ldGhldXMiLCJzdWIiOiJvYmplY3RzdG9yYWdlIn0.i1J-pXZ3s9_L4G-qWi9SGjcTKA_iTHZFM-CZXQgQIrkDWvWsvrpHONtcshZXb92QkgWNJhfmSAn6qqrjNQ6tvA
  metrics_path: /minio/prometheus/metrics
  scheme: http
  static_configs:
  - targets: [10.159.19.81]
```

Đây là cấu hình sẽ thêm vào prometheus để lấy metric từ minIO cluster (nếu dùng mode *public* thì bỏ trường **bearer_token** )

## 2. Cài đặt Promethus để thu thập metric
Thực hiện theo hướng dẫn tại: [Cài đặt Prometheus](https://github.com/longsube/ghichep-prometheus-v2/blob/master/docs/install_prometheus_container.md)

### 2.1. Cấu hình để Promethus lấy metric từ minIO cluster

Chỉnh sửa file cấu hình `/etc/prometheus/prometheus.yml` và thêm vào nội dung vừa lấy ở trên:
```sh
scrape_configs:
- job_name: minio-job
  metrics_path: /minio/prometheus/metrics
  scheme: http
  static_configs:
  - targets: [10.159.19.81]
```

### 2.2. Restart lại prometheus server
```sh
systemctl restart prometheus
```

### 2.3. Kiểm tra Prometheus đã có metric từ minIO (**minio_disks_total**). 

Truy cập: `http://10.159.19.85:9090/`
![minIO_1](../images/minIO_1.png)

## 3. Cài đặt Grafana để visualize metric

### 3.1. Cài đặt Grafana
```sh
docker run --restart=always -d -p 3000:3000 grafana/grafana
```

### 3.1. Truy cập vào dashboard của Grafana: 

`http://10.159.19.85:3000/`. `Tài khoản: admin, pass: admin` 

### 3.2. Thêm datasource của Prometheus
![minIO_8](../images/minIO_8.png)
Khai báo IP và port của host Prometheus, sau đó "Save and Test"
![minIO_9](../images/minIO_9.png)

### 3.3. Import file template sau để hiển thị đồ thị:
https://raw.githubusercontent.com/longsube/ghichep-minIO/master/tools/minIO_grafana_VNPTIT.json

![minIO_3](../images/minIO_3.png)

### 3.4. Kiểm tra giao diện đồ thị:
![minIO_2](../images/minIO_2.png)

*## 4. Giải thích các thông số giám sát (to be continued)*


## Tham khảo:

[1] - https://docs.min.io/docs/how-to-monitor-minio-using-prometheus.html

[2] - https://grafana.com/grafana/dashboards/11568

[3] - https://blog.minio.io/monitor-minio-server-with-prometheus-4ed537abcb74

[4] - https://grafana.com/docs/grafana/latest/installation/docker/#run-grafana

[5] - https://docs.min.io/docs/minio-client-complete-guide.html
