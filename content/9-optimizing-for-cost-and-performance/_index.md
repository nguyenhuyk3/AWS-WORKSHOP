---
title : "Tối ưu hóa chi phí và hiệu suất"
date : "`r Sys.Date()`"
weight : 9
chapter : false
pre : " <b> 9 </b> "
---

### Giới thiệu

Tối ưu hóa chi phí và hiệu suất là một phần thiết yếu trong vận hành hệ thống trên AWS, giúp bạn cân bằng giữa **khả năng mở rộng**, **tính sẵn sàng** và **ngân sách**. Trong chương này, chúng ta sẽ tập trung vào việc tối ưu cho các dịch vụ đã sử dụng trong hệ thống: **Amazon EC2**, **Amazon MSK**, **Amazon RDS**, và **Amazon ElastiCache**.

---

### Amazon EC2 – Linh hoạt theo nhu cầu

#### Chi phí
- **Sử dụng Auto Scaling Group** để chỉ khởi tạo số lượng instance cần thiết tùy theo tải (scale in/out).
- **Chọn đúng loại instance** (ví dụ: `t4g`, `m6g`, `c6i`...) dựa trên workload (compute/memory/storage-optimized).
- **Dùng Spot Instances** cho các workload không cần đảm bảo (batch job, worker...).
- **Tắt các instance nhàn rỗi** hoặc lên lịch dừng theo giờ hành chính với AWS Instance Scheduler.
- **Sử dụng Savings Plans hoặc Reserved Instances** nếu có workload cố định lâu dài.

#### Hiệu suất
- Cài đặt **CloudWatch Agent** để theo dõi CPU, Memory, Disk I/O.
- Dùng **Elastic Load Balancer (ELB)** để phân phối đều lưu lượng đến các instance.
- Cân nhắc sử dụng **Amazon EBS gp3** để tăng IOPS/dung lượng linh hoạt mà vẫn tiết kiệm.

---

### Amazon MSK – Kafka được quản lý

#### Chi phí
- **Chọn loại instance phù hợp** cho broker (ví dụ: kafka.t3.small cho môi trường dev/test).
- Giảm số lượng broker khi tải nhẹ và không cần khả năng chịu lỗi cao.
- **Tắt các cluster không dùng** hoặc chuyển sang môi trường serverless nếu workload không liên tục.
- **Sử dụng Kafka Topic Retention Policy hợp lý** để giới hạn dung lượng lưu trữ (ví dụ: chỉ giữ 3 ngày).

#### Hiệu suất
- **Bật Enhanced Monitoring** để theo dõi chi tiết throughput, replication lag.
- Tối ưu cấu hình Kafka như `batch.size`, `linger.ms` cho producer và `fetch.min.bytes` cho consumer.
- Tránh Under-replicated partitions bằng cách phân bổ phân vùng đều giữa các broker.
- Dùng **CloudWatch Dashboard** để theo dõi real-time tình trạng cluster.

---

### Amazon RDS – Cơ sở dữ liệu quan trọng

#### Chi phí
- **Chọn đúng loại instance** (db.t4g, db.m6g cho workloads vừa phải).
- Dùng **Aurora Serverless v2** nếu tải không liên tục.
- **Tự động dừng RDS khi không dùng** (cho môi trường dev/test).
- **Xóa snapshots không cần thiết** để tiết kiệm chi phí lưu trữ.
- Sử dụng **Reserved Instances** nếu có nhu cầu chạy dài hạn.

#### Hiệu suất
- Bật **Performance Insights** để xác định truy vấn chậm, lock, wait events...
- Tối ưu **index và query** thường xuyên dựa trên slow query log.
- Thiết lập **Read Replicas** để phân tải đọc, giảm áp lực node chính.
- Theo dõi các chỉ số như `ReadLatency`, `WriteLatency`, `DatabaseConnections`.

---

### Amazon ElastiCache – Cache hiệu quả, chi phí hợp lý

#### Chi phí
- **Chọn đúng engine và kích thước node** (`cache.t4g.small` cho test/dev, `cache.m6g.large` cho production).
- Giảm số lượng node trong cụm nếu cache hit ratio cao và tải nhẹ.
- **Không bật Multi-AZ nếu không cần thiết** (đối với workload không yêu cầu HA).
- Xem xét thời gian TTL của keys để tránh giữ dữ liệu quá lâu gây tốn bộ nhớ.

#### Hiệu suất
- Theo dõi `CacheHitRate`, `Evictions`, `CurrConnections` từ CloudWatch.
- Tối ưu TTL phù hợp để tránh eviction sớm hoặc giữ cache dư thừa.
- Dùng **cluster mode** để scale horizontal nếu cần hiệu suất cao.
- Dùng **Redis slowlog** để phát hiện các truy vấn tốn thời gian.

---

### Tổng kết

Việc tối ưu hóa không chỉ giúp giảm chi phí mà còn cải thiện hiệu suất hệ thống đáng kể. Một số nguyên tắc chung:

- **Giám sát liên tục** để xác định điểm nghẽn và lãng phí.
- **Tự động hóa tắt/mở tài nguyên** khi không sử dụng.
- **Chọn đúng công cụ và kiến trúc phù hợp với từng loại workload**.

Hãy xem việc tối ưu chi phí như một quá trình liên tục, đi đôi với sự phát triển và mở rộng của hệ thống.

