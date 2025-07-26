---
title : "Giám sát và khả năng quan sát với Amazon CloudWatch"
date : "`r Sys.Date()`"
weight : 8
chapter : false
pre : " <b> 8 </b> "
---

### Amazon CloudWatch là gì?

Amazon CloudWatch là dịch vụ quan sát được quản lý hoàn toàn, cung cấp dữ liệu và thông tin chi tiết hữu ích cho tài nguyên, ứng dụng và dịch vụ AWS. Dịch vụ này cho phép bạn:

- Thu thập nhật ký và số liệu từ các dịch vụ AWS và ứng dụng tùy chỉnh
- Tạo cảnh báo dựa trên ngưỡng hoặc phát hiện bất thường
- Trực quan hóa dữ liệu bằng bảng điều khiển
- Tự động phản hồi các thay đổi bằng sự kiện và quy tắc

CloudWatch đóng vai trò trung tâm trong việc chẩn đoán các sự cố vận hành và tối ưu hóa hiệu suất trong kiến trúc phân tán, hướng sự kiện hoặc không máy chủ.

---

### Tại sao nên sử dụng Logging và CloudWatch?

Việc thiết lập chế độ ghi nhật ký và khả năng quan sát phù hợp với CloudWatch là rất cần thiết vì những lý do sau:

1. **Khắc phục sự cố và Gỡ lỗi:**
Nhật ký giúp bạn theo dõi các vấn đề đến từng chức năng hoặc sự kiện cụ thể, cho phép bạn giải quyết lỗi nhanh hơn.

2. **Giám sát Hoạt động:**
Các số liệu cho phép bạn theo dõi tình trạng hệ thống — bao gồm tỷ lệ gọi hàm, độ trễ và tần suất lỗi.

3. **Phát hiện Lỗi:**
Các cảnh báo sẽ chủ động thông báo cho bạn về các vấn đề như tồn đọng tin nhắn trong hàng đợi, điều tiết hoặc tràn DLQ.

4. **Kiểm toán và Tuân thủ:**
Nhật ký đóng vai trò là bản ghi lịch sử về hành vi của hệ thống để theo dõi kiểm toán và đánh giá sau sự cố.

5. **Tối ưu hóa Hiệu suất:**
Các số liệu của CloudWatch có thể được sử dụng để phân tích các điểm nghẽn, tối ưu hóa việc sử dụng máy tính/bộ nhớ và giảm chi phí.

---

### Mục tiêu

Thiết lập hệ thống giám sát tập trung (centralized monitoring) và khả năng quan sát (observability) cho các thành phần của ứng dụng không máy chủ (serverless) bằng cách tận dụng Amazon CloudWatch để thu thập logs, metrics và sự kiện từ nhiều dịch vụ khác nhau trong hệ sinh thái AWS.

Các thành phần được giám sát:

1. Amazon EC2

- **Metrics**
  * CPU, Memory, Disk I/O, Network In/Out (qua CloudWatch Agent).
- **Logs**
  * Application logs (ví dụ: logs từ ứng dụng Node.js, Go, Java...).
- **System logs**
  * /var/log/messages, /var/log/syslog, v.v.
- **Cách triển khai**:
  * Cài CloudWatch Agent trên EC2 instance.
  * Cấu hình thu thập cả metrics và logs, gửi về CloudWatch.

2. Amazon MSK (Managed Kafka)

- **Metrics**: Broker-level (CPU, memory, disk).
  * Kafka-level (bytes in/out, replication lag, under-replicated partitions).
- **Logs**
  * Kafka broker logs (controller logs, server logs…).
- **Cách triển khai**:
  * Bật enhanced monitoring và log delivery về CloudWatch Logs khi tạo MSK Cluster hoặc cập nhật sau.
  * Dùng các biểu đồ và alarm để theo dõi tình trạng cluster.

3. Amazon RDS (PostgreSQL, MySQL, Aurora...)
- **Metrics**
  * CPU, FreeableMemory, DatabaseConnections, Read/Write Latency.
- **Logs**
  * Error log, General log, Slow query log.
- **Cách triển khai**:
  * Bật tính năng gửi logs về CloudWatch Logs trong cấu hình RDS.
  * Sử dụng CloudWatch Dashboard để hiển thị các metric quan trọng.

4. Amazon ElastiCache (Redis/Memcached)
- **Metrics**
  * CPUUtilization, CurrConnections, Evictions, CacheHits/Misses, ReplicationLag.
- **Logs**
  * ❌ Không hỗ trợ CloudWatch Logs trực tiếp (Redis logs không tự gửi lên được).
- **Cách triển khai**:
  * Sử dụng các metric sẵn có trong CloudWatch để tạo dashboard và cảnh báo.
  * Nếu cần log chi tiết như Redis Slowlog, phải thu thập thủ công.

---

## Các thành phần kiến trúc đang được quan sát

Các thành phần AWS sau đây cần được tích hợp với CloudWatch:

| **Thành phần**         | **Các số liệu và nhật ký cần giám sát**                                                                                                                                  |
|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Amazon EC2**         | - `CPUUtilization`, `MemoryUsage`, `Disk I/O`, `Network I/O`<br>- Application logs<br>- System logs (`/var/log/messages`, `/var/log/syslog`)                           |
| **Amazon MSK**         | - `BytesInPerSec`, `BytesOutPerSec`, `UnderReplicatedPartitions`, `ReplicationLag`<br>- Kafka broker logs (controller, server, state-change logs)                      |
| **Amazon RDS**         | - `CPUUtilization`, `FreeableMemory`, `ReadLatency`, `WriteLatency`, `DatabaseConnections`<br>- Error logs, Slow query logs, General logs                              |
| **Amazon ElastiCache** | - `CPUUtilization`, `CurrConnections`, `Evictions`, `CacheHits/Misses`, `ReplicationLag`<br>- ❌ *Không hỗ trợ gửi logs trực tiếp về CloudWatch*                        |

---

### Xây dựng bảng điều khiển CloudWatch tập trung

Tạo bảng thông tin tùy chỉnh trong (CloudWatch → Bảng điều khiển) với các tiện ích sau:

| **Widget Title**                     | **Metrics Displayed**                                                                                   |
|-------------------------------------|----------------------------------------------------------------------------------------------------------|
| EC2 System Health                   | `CPUUtilization`, `MemoryUsage`, `DiskReadBytes`, `DiskWriteBytes`, `NetworkIn`, `NetworkOut`           |
| EC2 Application Logs                | Log stream từ các ứng dụng đang chạy trên EC2 (theo thời gian thực)                                     |
| RDS Performance                     | `CPUUtilization`, `FreeableMemory`, `ReadLatency`, `WriteLatency`, `DatabaseConnections`                |
| RDS Error & Slow Queries            | Error log & Slow query log (truy xuất từ CloudWatch Logs Insights)                                      |
| MSK Cluster Throughput              | `BytesInPerSec`, `BytesOutPerSec`, `UnderReplicatedPartitions`, `ReplicationLag`                        |
| MSK Broker Logs                     | Kafka broker logs (controller, server, state-change logs)                                               |
| ElastiCache Usage & Replication     | `CPUUtilization`, `CurrConnections`, `CacheHits`, `CacheMisses`, `ReplicationLag`                       |
| Application Errors (Across Services)| Truy vấn lỗi HTTP 4xx/5xx từ CloudWatch Logs Insights (có thể áp dụng cho API Gateway, Lambda, EC2...) |
| Tổng quan Hệ thống                  | Biểu đồ tổng hợp trạng thái hoạt động của tất cả thành phần (OK/ALARM/INSUFFICIENT)                     |

Sử dụng các hình ảnh trực quan này để có cái nhìn tổng quan theo thời gian thực về tình trạng và hiệu suất của hệ thống.

---

### Kết quả mong đợi

Sau khi triển khai các bước trên:

- Tất cả các thành phần ứng dụng liên tục truyền nhật ký đến CloudWatch.
- Cảnh báo phát hiện và cảnh báo về lỗi hệ thống và độ trễ.
- Bảng điều khiển cho phép phân tích hiệu suất và độ tin cậy theo thời gian thực.
- Nhật ký và số liệu cho phép gỡ lỗi hiệu quả, phân tích nguyên nhân gốc rễ và giám sát hoạt động.
