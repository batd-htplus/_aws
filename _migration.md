# T-SCRUM AWS Migration: Giải Pháp Tích Hợp Database

> **Mục tiêu**: Giải quyết các vấn đề tích hợp database khi migrate **DB Core (tvmk06)** sang PostgreSQL trên AWS, trong khi **DB P+ / PLASURM+ (tvmk02, tvmk03, tvmk05)** vẫn được **giữ nguyên hoàn toàn trên on-premises**.

**Mapping với source hiện tại (dựa trên sơ đồ AS-IS):**

| Thành phần          | Trong repo / thực tế hiện tại                                      |
|---------------------|---------------------------------------------------------------------|
| Ứng dụng            | Java + Tomcat + yhFramework (servlets, online batch, scheduler)   |
| DB Core             | **`tvmk06`** (Core System) → Migrate toàn bộ lên PostgreSQL trên AWS |
| DB P+ / PLASURM+    | **`tvmk02`** (Plasurm+ chính), tvmk03 (FW/Common), tvmk05 (Work/Batch) → **Giữ nguyên** trên SQL Server on-premises |
| Vai trò tvmk06 sau migrate | Chỉ còn là **DB replica/read-side on-prem** nhận đồng bộ từ AWS (không còn là DB xử lý chính) |

**Nguyên tắc thiết kế quan trọng:**
- DB Core (tvmk06) sẽ được migrate và vận hành hoàn toàn trên AWS PostgreSQL.
- tvmk06 on-prem **không bị xóa ngay**, chỉ thay đổi role thành DB replica nhận đồng bộ một chiều từ AWS PostgreSQL.
- DB P+ (tvmk02 và các DB liên quan) vẫn giữ nguyên logic hiện tại và tiếp tục truy vấn qua DB link/synonym sang `tvmk06`.

**Một dòng kiến trúc tổng quát (final):**
`AWS Core PostgreSQL (source of truth) -> CDC Runtime on AWS -> tvmk06 SQL Server 2012 (replica sink) -> tvmk02 query via DB link/synonym`

---

## PHẦN 1: GIẢI PHÁP TÍCH HỢP DATABASE

### Giải Pháp 1: Debezium CDC + Change Stream (Đồng bộ DB Core từ AWS xuống tvmk06)

**Mục đích**: Giữ cơ chế Linked Server + Synonym hiện tại cho PLASURM, đồng thời đổi nguồn dữ liệu chính sang AWS.

**Kiến trúc cốt lõi**:
- DB Core mới → **Migrate full** sang RDS PostgreSQL trên AWS và trở thành **source of truth**.
- `tvmk06` on-prem → chuyển thành **read replica / integration-side DB** để `tvmk02` tiếp tục truy vấn như cũ.
- Sử dụng CDC pipeline với **Debezium PostgreSQL Source Connector** + Kafka + SQL Server sink để đồng bộ một chiều từ AWS về `tvmk06`.

**Quy trình hoạt động chi tiết:**
1. Bật logical replication/CDC trên PostgreSQL AWS cho các bảng Core cần publish.
2. **Debezium Source Connector** thực hiện **Initial Snapshot** (full load) từ AWS Core.
3. Debezium connector liên tục capture thay đổi (INSERT/UPDATE/DELETE) và đẩy events vào Amazon MSK (Kafka).
4. Sink connector nhận events và **upsert** vào SQL Server `tvmk06` (schema tương thích với synonym hiện tại).
5. PLASURM (`tvmk02`) tiếp tục truy vấn `tvmk06` qua DB link/synonym như trước, không phải đổi lớn ở tầng ứng dụng cũ.

**Lợi ích chính:**
- AWS Core trở thành nguồn dữ liệu chính, tách khỏi ràng buộc on-prem.
- Giữ nguyên cơ chế truy vấn cũ của PLASURM qua `tvmk02` -> `tvmk06`.
- Giảm rủi ro sửa code lớn ở hệ thống legacy trong giai đoạn chuyển tiếp.
- Có thể mở rộng event stream cho audit, reconciliation, observability.

**Rủi ro & Biện pháp giảm thiểu:**
| Rủi ro                              | Mức độ | Biện pháp giảm thiểu                              |
|-------------------------------------|--------|---------------------------------------------------|
| Lag CDC > 5 giây                    | Trung bình | Giám sát Kafka lag qua CloudWatch + alert tự động |
| Schema PostgreSQL hoặc schema replica tvmk06 thay đổi | Cao    | Áp dụng schema governance + migration kiểm soát tương thích |
| Initial Snapshot chậm do data lớn   | Thấp   | Thực hiện ngoài giờ cao điểm, chia batch         |

**Chi phí phát sinh ước tính (tháng):**
- Amazon MSK (small cluster) + connector runtime: **$48 – $100**
- **Tổng thêm cho CDC/replication**: khoảng **$60/tháng**

---

### Giải Pháp 2: S3 + SQS + Integration Agent (Thay thế cơ chế Shared Folder)

**Quy trình mới** (DB P+ vẫn giữ nguyên, tvmk06 là DB replica/read-side):
1. Core App (AWS) tạo file CSV.
2. Upload CSV + checksum `.sha256` lên Amazon S3.
3. Gửi message qua Amazon SQS.
4. Integration Agent (on-prem) nhận message qua **SQS long polling**, tải file từ S3, kiểm tra checksum và ghi atomic vào thư mục cục bộ.
5. PLASURM+ (tvmk02) tiếp tục import như cũ.

**Lợi ích**: Giữ nguyên logic PLASURM+ mà không cần mount shared folder từ AWS.

**Rủi ro & Biện pháp giảm thiểu:**
| Rủi ro                              | Mức độ | Biện pháp giảm thiểu                              |
|-------------------------------------|--------|---------------------------------------------------|
| File CSV hỏng khi truyền            | Trung bình | Kiểm tra checksum SHA256 bắt buộc                 |
| Integration Agent downtime          | Cao    | Chạy dưới dạng service với auto-restart           |

**Chi phí phát sinh ước tính (tháng):**
- S3 + SQS: **$12 – $20**
- **Tổng thêm**: khoảng **$16/tháng**

---

## PHẦN 2: TÓM TẮT CHI PHÍ PHÁT SINH TOÀN BỘ

| Hạng mục                  | Chi phí ước tính / tháng (USD) |
|---------------------------|--------------------------------|
| Debezium CDC (MSK)        | 60                             |
| S3 + SQS + Agent          | 16                             |
| **Tổng chi phí phát sinh** | **≈ 76 USD/tháng**            |

**Lưu ý**: Chi phí có thể giảm thêm khi dùng MSK Serverless hoặc tối ưu instance.

---

## KẾT LUẬN

Với thiết kế này:
- **DB Core chính** được migrate và vận hành hoàn toàn trên AWS PostgreSQL.
- `tvmk06` on-prem chuyển thành **replica/read-side DB** nhận đồng bộ từ AWS.
- **DB P+ (tvmk02, tvmk03, tvmk05)** giữ nguyên logic và tiếp tục truy vấn qua DB link/synonym hiện có.
- Toàn bộ tích hợp được xử lý an toàn, hiệu suất cao, có cơ chế retry và giám sát rõ ràng.

Chúng tôi sẵn sàng đính kèm sơ đồ AS-IS (hình bạn gửi), thực hiện PoC hoặc điều chỉnh thêm theo yêu cầu.

# Tổng quan:

**Biểu đồ tổng quan (diagram-as-code)**
![AWS](diagram-export-4-15-2026-9_18_48-PM.png)


```text
// On-Premises Environment
"On-Premises" [color: gray, icon: server] {

  // Application Layer (On-Prem)
  On-Prem Application [icon: layers] {
    Tomcat PLASURM [icon: server, label: "PLASURM"]
  }

  DB Core Replica [icon: database] {
    "tvmk06 On-Prem" [icon: azure-sql-database, label: "tvmk06 (Replica Bridge for P+)"]
  }

  DB P+ PLASURM+ [icon: database, color: blue] {
    tvmk02 [icon: azure-sql-database, label: "tvmk02 (Plasurm+)"]
    tvmk03 [icon: azure-sql-database, label: "tvmk03 (FW/Common)"]
    tvmk05 [icon: azure-sql-database, label: "tvmk05 (Work/Batch)"]
  }

  Integration Agent [icon: cpu, label: "Integration Agent (SQS long polling)"]
  Local Shared Folder [icon: folder]
}

// AWS Cloud Environment
AWS Cloud [color: orange, icon: aws] {
  Application Layer [icon: layers] {
    PHP App [icon: php, label: "Laravel App"]
    CSV Export Job [icon: file, label: "CSV Export Job (cron daily 01:00)"]
  }

  RDS PostgreSQL [icon: aws-rds] {
    Core Schema [icon: aws-aurora, label: "core schema"]
  }

  CDC Runtime [color: green, icon: activity] {
    Debezium Source [icon: aws-msk, label: "Debezium PostgreSQL Source"]
    MSK Kafka [icon: aws-msk, label: "Amazon MSK"]
    Sink Connector [icon: aws-msk, label: "SQL Server Sink"]
  }

  File Transfer [color: purple, icon: file] {
    S3 Bucket [icon: aws-s3, label: "S3 (CSV + Checksum)"]
    SQS Queue [icon: aws-sqs, label: "SQS Queue"]
  }
}

// Core -> PLASURM one-way replication
Core Schema --> Debezium Source: capture changes in AWS
Debezium Source > MSK Kafka: change events
MSK Kafka > Sink Connector: consume events
Sink Connector > "tvmk06 On-Prem": upsert replica

// Legacy query path
tvmk02 <> "tvmk06 On-Prem": query via DB link/synonym
On-Prem Application <> tvmk02
tvmk02 <> tvmk03
tvmk02 <> tvmk05

// File transfer path
PHP App > CSV Export Job
CSV Export Job > S3 Bucket
S3 Bucket > SQS Queue: notify
Integration Agent <-- SQS Queue: receive by long polling (event-driven)
Integration Agent > S3 Bucket: download files
Integration Agent > Local Shared Folder: checksum verify + atomic write
Local Shared Folder > tvmk02: import CSV after validation
```

**Các điểm bắt buộc phải thể hiện trên biểu đồ Eraser**
- Luồng chính: `AWS core(PostgreSQL) -> Debezium CDC Runtime(AWS) -> tvmk06(SQL Server 2012 replica bridge)`.
- Luồng truy vấn legacy: `tvmk02/PLASURM -> DB link/synonym -> tvmk06`.
- Luồng CSV một chiều: `CSV Export Job (daily 01:00) -> S3 + SQS -> Integration Agent (SQS long polling) -> checksum verify + atomic write -> tvmk02 import`.
- Không sử dụng Outbox trong phạm vi thiết kế hiện tại để giảm độ phức tạp.

**Quy tắc vận hành đi kèm biểu đồ**
- `core` trên AWS là **source of truth** cho domain Core.
- `tvmk06` là **compatibility bridge** cho PLASURM đọc theo cơ chế cũ.
- Account nghiệp vụ on-prem phải **read-only** trên vùng bảng replicated ở `tvmk06`.
- Chỉ replication sink account được phép ghi vào vùng replicated trên `tvmk06`.

---

## PHẦN 3: ĐIỀU KIỆN CẦN & RISK CHO PHƯƠNG ÁN GIỮ DB LINK

> **Phương án áp dụng**: Giữ cơ chế DB Link/Synonym hiện tại cho PLASURM (`tvmk02` -> `tvmk06`), nhưng đổi vai trò `tvmk06` thành DB replica nhận dữ liệu một chiều từ AWS PostgreSQL.

### 3.0 CDC nằm ở đâu? SQL Server 2012 cần cài gì?

- CDC/replication logic nằm ở **AWS runtime** (Debezium Source + MSK + Sink Connector), không chạy trong SQL Server 2012.
- `tvmk06` SQL Server 2012 là **đích nhận dữ liệu (sink)**, không phải nguồn CDC trong phương án này.
- SQL Server 2012 **không cần bật SQL Server CDC** cho phương án AWS -> `tvmk06`.
- SQL Server 2012 **không cần cài Debezium** hoặc plugin CDC trong DB engine.
- Việc cần làm trên `tvmk06`: mở port kết nối, cấp quyền ghi cho user sink, đảm bảo PK/index/upsert key, theo dõi lock và transaction log.

### 3.1 Điều kiện cần để triển khai

| Nhóm điều kiện | Điều kiện bắt buộc | Ghi chú triển khai |
|----------------|--------------------|--------------------|
| Kết nối mạng AWS -> On-Prem | Có đường kết nối riêng (Site-to-Site VPN hoặc Direct Connect) từ VPC AWS về DC on-prem | Không dùng public Internet cho kết nối DB |
| Firewall / Port | Mở chiều AWS -> On-Prem cho các cổng cần thiết: `1433/TCP` (SQL Server `tvmk06`), `9092/TCP` (nếu self-managed Kafka), `443/TCP` (AWS API/S3/SQS/CloudWatch) | Chỉ allow theo source CIDR/SG cụ thể, không mở rộng toàn mạng |
| DNS / Name resolution | AWS resolve được host on-prem (ví dụ `tvmk06`) và on-prem resolve được endpoint AWS cần dùng | Bắt buộc có DNS forwarding hoặc host mapping chuẩn |
| Runtime Integration Agent (on-prem) | Cài AWS SDK/client cho S3 + SQS, bật TLS, cấu hình SQS long polling | Khuyến nghị `WaitTimeSeconds=20` để giảm số lần quét |
| IAM quyền tối thiểu cho Agent | Cấp `sqs:ReceiveMessage/DeleteMessage/ChangeMessageVisibility` và `s3:GetObject` (thêm `ListBucket` nếu cần) | Dùng credential ngắn hạn/rotation, không hardcode access key |
| Cấu hình SQL Server `tvmk06` | Bật chế độ nhận upsert từ pipeline (JDBC sink/agent), tối ưu index theo key đồng bộ, đủ log file | Kiểm soát lock escalation để tránh ảnh hưởng truy vấn từ `tvmk02` |
| Cấu hình PostgreSQL AWS | Bật logical replication/CDC theo công nghệ chọn, có user chỉ đọc CDC stream | Tách quyền rõ ràng: user app và user CDC |
| Giữ tương thích schema | Bảng/cột/index/collation trên `tvmk06` phải giữ contract cũ để synonym/query từ `tvmk02` chạy được | Đây là điều kiện sống còn để không sửa nhiều code PLASURM |
| Phân quyền DB Link/Synonym | `tvmk02` vẫn truy vấn được object synonym trỏ sang `tvmk06`; không đổi tên object đột ngột | Chuẩn bị script verify sau cutover |
| Quan trắc & cảnh báo | Có dashboard cho replication lag, error rate, throughput, dead-letter events | Cần cảnh báo theo ngưỡng (VD lag > 5s, > 30s, > 5m) |
| Quy trình vận hành | Có runbook: restart connector, re-sync từng bảng, rollback về snapshot trước cutover | Có đầu mối trách nhiệm rõ (Infra/DBA/App) |

### 3.2 Danh sách risk cần xử lý trước Go-Live

| Risk | Mức độ | Tác động | Biện pháp xử lý |
|------|--------|----------|-----------------|
| PLASURM vẫn ghi vào các bảng đang coi là replica (`TM_KWSRT`, `TW_SGJSKRNKI`, `TW_SYHJSKRNKI`, `TT_TVMKURGSWK`...) | Rất cao | Xung đột dữ liệu hai chiều, mất nhất quán | Khóa chức năng ghi tương ứng hoặc tách riêng bảng ghi nghiệp vụ trước cutover |
| Replication lag cao giờ cao điểm | Cao | `tvmk02` đọc dữ liệu trễ so với Core AWS | Thiết lập SLA lag, autoscale connector, tăng partition/throughput, cảnh báo sớm |
| Mismatch kiểu dữ liệu/collation PostgreSQL -> SQL Server | Cao | Lỗi insert/update, sai so sánh chuỗi tiếng Nhật | Định nghĩa mapping chuẩn từng cột, test full regression theo bảng trọng yếu |
| Mất kết nối AWS <-> On-Prem | Cao | Replica ngừng cập nhật, backlog tăng | Thiết kế retry + backoff, queue buffer, cảnh báo NOC, runbook failover |
| Lỗi thứ tự event (out-of-order) | Trung bình | Dữ liệu cuối cùng sai trạng thái | Upsert idempotent theo business key + version/timestamp |
| Schema thay đổi từ Core AWS nhưng chưa rollout xuống `tvmk06` | Cao | Connector lỗi hàng loạt, dừng đồng bộ | Quản trị schema bằng change pipeline bắt buộc (migrate script + kiểm tra tương thích) |
| Tắc nghẽn/lock trên `tvmk06` khi vừa đọc nhiều từ `tvmk02` vừa ghi replicate | Cao | Query chậm, timeout nghiệp vụ | Tuning index, batch size, isolation level; tách khung giờ bulk load |
| Không có cơ chế đối soát dữ liệu định kỳ | Cao | Sai lệch âm thầm khó phát hiện | Chạy reconciliation hằng ngày (count/checksum theo partition key) |
| Quyền truy cập rộng quá mức | Trung bình | Rủi ro bảo mật và thao tác nhầm | Principle of least privilege, rotate secret định kỳ, audit truy cập |
| Thiếu kế hoạch rollback | Rất cao | Sự cố kéo dài, ảnh hưởng nghiệp vụ | Chuẩn bị sẵn snapshot/mốc dữ liệu + kịch bản quay lui trong 30-60 phút |
