# Bài tập: Tách index theo từng service bằng biến động trong Logstash output

**Ngày thực hiện:** 07/11/2026
**Dự án:** StoreX (Spring Boot 3.3, Java 17) gồm `user-service` và `order-service`, dùng chung PostgreSQL
**Môi trường chạy:** AWS EC2 (Ubuntu, instance c7i-flex.large), toàn bộ chạy bằng Docker Compose

---

## 1. Cấu hình gốc và vấn đề

### Tình huống

Hệ thống có `user-service` và `order-service`. Log của cả hai service đều bị đẩy chung vào một index. Quản trị viên muốn tách riêng index theo tên ứng dụng.

### Cấu hình gốc (output block của `logstash.conf`)

```
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"   # Lỗi: tên index cố định, không phụ thuộc app
  }
}
```

Trong môi trường thực hành, `output` chưa đặt `index` nên Logstash dùng tên mặc định `logstash-...`. Kết quả tương đương: mọi log đều vào một index chung.

### Vấn đề

Tên index chỉ phụ thuộc vào ngày, không phụ thuộc vào service. Khi kiểm tra:

```bash
curl -s 'localhost:9200/_cat/indices?v'
```

Chỉ có một index của Logstash là `logstash-2026.11.07-000001`, chứa log của cả hai service. Hệ quả:

- Không phân quyền, giữ log hay xóa log riêng cho từng service được.
- Mọi truy vấn phải lọc thêm theo field `app_name`, trên một index ngày càng lớn.
- Không áp dụng được chính sách lưu trữ khác nhau cho từng service (ví dụ giữ log `order-service` lâu hơn).

---

## 2. Yêu cầu và cách thực hiện

### Yêu cầu

Giả sử Logback gửi kèm trường `app_name` trong chuỗi JSON. Sửa biến `index` trong Logstash thành `%{[app_name]}-%{+YYYY.MM.dd}`.

### Điều kiện cần có

Hai service gửi mỗi log là một JSON có sẵn `app_name` (`user-service` hoặc `order-service`) nhờ `customFields` trong `LogstashEncoder`:

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <customFields>{"app_name":"${APP_NAME}","env":"${APP_ENV}"}</customFields>
</encoder>
```

Input `tcp` của Logstash dùng `codec => json_lines` (bài trước) nên JSON đã được parse thành field riêng. Điều này là bắt buộc: với codec `plain`, `app_name` nằm trong chuỗi `message` và không dùng được trong `%{[app_name]}`.

### Bước 1. Sửa `logstash.conf`

```
# logstash.conf
input {
  tcp {
    port => 5000
    codec => json_lines
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "%{[app_name]}-%{+YYYY.MM.dd}"
  }
}
```

Giải thích cú pháp:

| Thành phần | Ý nghĩa |
|---|---|
| `%{[app_name]}` | Lấy giá trị field `app_name` của từng event. |
| `%{+YYYY.MM.dd}` | Lấy ngày từ `@timestamp` của event, mỗi ngày một index mới. |

Áp dụng cấu hình bằng cách khởi động lại Logstash:

```bash
docker compose restart logstash
docker compose logs logstash | grep -i "pipeline started\|Successfully started"
```

### Bước 2. Sinh log từ cả hai service

```bash
curl -s localhost:8081/api/users > /dev/null
curl -s localhost:8081/api/users/999 > /dev/null
curl -s -X POST localhost:8082/api/orders -H 'Content-Type: application/json' \
  -d '{"userId":1,"productName":"Chuot","quantity":1,"amount":350000}' > /dev/null
sleep 10
```

### Bước 3. Kiểm tra các index

```bash
curl -s 'localhost:9200/_cat/indices?v'
```

---

## 3. Kết quả

Kết quả `GET _cat/indices?v` trên Elasticsearch:

```text
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .geoip_databases                _UJb0slIRzaoxdlcQ290Ug   1   0         33            0     31.5mb         31.5mb
green  open   .kibana_7.17.0_001              PjyLjZcFQg-NCyVZAD4jhw   1   0         15            0      2.3mb          2.3mb
green  open   .apm-custom-link                u-7z49e4RJygEAecYegk4g   1   0          0            0       226b           226b
green  open   .apm-agent-configuration        -njP1DeLR-iAhhSHkAzTWA   1   0          0            0       226b           226b
yellow open   user-service-2026.11.07         I4T600eoQGmcCgEP_-UfwQ   1   1          4            0     12.8kb         12.8kb
green  open   .kibana_task_manager_7.17.0_001 ERGnrnrES3Gtdv2lqwZtSg   1   0         17         1407      303kb          303kb
yellow open   order-service-2026.11.07        EAPOBXcwSKSghr9w5AmudQ   1   1          3            0     12.7kb         12.7kb
yellow open   logstash-2026.11.07-000001      6WC0YSRoQVO9h4wiaHgspQ   1   1         84            0       84kb           84kb
```

### Nhận xét

- Đạt yêu cầu của đề: xuất hiện hai index riêng `user-service-2026.11.07` (4 document) và `order-service-2026.11.07` (3 document), đúng định dạng `<app_name>-<ngày>`.
- Mỗi service có index riêng nên có thể tạo index pattern riêng trên Kibana (`user-service-*`, `order-service-*`), phân quyền riêng và áp dụng chính sách lưu trữ riêng cho từng service.
- Khi thêm một service mới chỉ cần đặt `app_name` đúng, Logstash tự tạo index mới mà không phải sửa `logstash.conf`.
- Index cũ `logstash-2026.11.07-000001` (84 document) vẫn còn, chứa dữ liệu từ trước khi sửa. Dữ liệu cũ không được tách lại, chỉ log mới mới vào index riêng.
- Các index hiện `yellow` vì Elasticsearch mặc định tạo 1 replica nhưng môi trường chỉ có một node. Không ảnh hưởng chức năng; muốn `green` thì đặt `number_of_replicas: 0`.

### Lưu ý và hướng mở rộng

- **Tên index phải viết thường.** `app_name` có chữ hoa sẽ khiến Elasticsearch từ chối tạo index.
- **Event không có `app_name`** (ví dụ gửi tay bằng `nc` một dòng JSON thiếu field này) sẽ tạo index tên `%{[app_name]}-2026.11.07` nguyên văn. Nên thêm giá trị mặc định ở `filter`:
  ```
  filter {
    if ![app_name] { mutate { add_field => { "app_name" => "unknown" } } }
  }
  ```
- Nếu tên index xuất hiện hậu tố như `-000001` thay vì ngày, nguyên nhân là ILM của Logstash 7.17; thêm `ilm_enabled => false` trong `elasticsearch { }`.
- Có thể tách thêm theo môi trường: `index => "%{[app_name]}-%{[env]}-%{+YYYY.MM.dd}"`.
- Với số lượng service lớn, mỗi ngày một index cho mỗi service sinh ra rất nhiều shard nhỏ. Khi đó cân nhắc dùng ILM kèm index template, hoặc đổi sang chu kỳ theo tuần.
