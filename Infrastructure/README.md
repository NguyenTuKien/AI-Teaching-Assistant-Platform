# 1. Tìm hiểu Ops và cách chia cluster
Do hệ thống thiên về AI nên tải sẽ không đều. Web/API có thể scale ngang, nhưng database, vector database, object storage và queue phải được thiết kế ổn định hơn. Tim hiểu cách chia cluster thành các lớp:
- Edge layer: load balancer, reverse proxy, rate limiting.
- Web/API layer: scale ngang nhiều replica.
- AI worker layer: xử lý LLM, tutor, guardrails.
- RAG worker layer: parse tài liệu, embedding, indexing.
- Data layer: PostgreSQL, vector DB, object storage, Redis/queue.
- CI/autograding layer: tách riêng để việc chạy test code sinh viên không ảnh hưởng hệ thống chính.
- Monitoring layer: log, metrics, tracing, alert.

Mục tiêu là xác định service nào cần scale ngang, service nào cần chạy ổn định/high availability, service nào nên xử lý bất đồng bộ qua queue.

## 1.1. Context
- Giả định hệ thống chạy trên server, chỉ giới hạn cho sinh viên trong trường sử dụng.
- Ứng tính quy mô:
    - Khi cao điểm có thể có khoảng 2000-3000 user active, 200-300 req/s.
    - Database: khoảng vài chục GB, chủ yếu là tài liệu, embedding và user info.
    - Trước mắt triển khai trên 3 môn học pivot là Học máy, Học sâu và Khai phá dữ liệu lớn.
- Dự tính triển khai chính trên Kubernetes vì khả năng co giãn và quản lý tài nguyên tốt.

## 1.2. Kiến trúc hệ thống
### 1.2.1. Layer 1: Tầng mạng
```mermaid
graph TB
    Internet(["🌐 Internet\n(Users / Clients)"])

    subgraph CF["☁️ Cloudflare"]
        direction TB
        DNS["🔗 Domain (DNS)"]
        SSL["🔒 SSL/TLS Termination"]
        DDOS["🛡️ DDoS Protection"]
        CDN["⚡ CDN / Edge Cache"]
        Tunnel["🚇 Cloudflare Tunnel\n(cloudflared)"]

        DNS --> SSL --> DDOS --> CDN --> Tunnel
    end

    subgraph K8S["☸️ Kubernetes Cluster"]
        direction TB

        subgraph GW["API Gateway (K8s Service - ClusterIP)"]
            RATE["🚦 Rate Limiting"]
            AUTH["🔑 Auth / Routing"]
        end

        subgraph PODS["Microservices"]
            P1["📦 Service A"]
            P2["📦 Service B"]
            P3["📦 Service N"]
        end

        RATE --> AUTH
        AUTH --> P1 & P2 & P3
    end

    Internet --> CF
    Tunnel -->|"Encrypted Tunnel\n(trỏ thẳng vào ClusterIP)"| RATE

```

### 1.2.2. Layer 2: Tầng ứng dụng

#### Phía Frontend
Chia làm 3 site độc lập ứng với 3 vai trò (role):
*   **Admin**: Quản trị hệ thống, quản lý môn học, người dùng, phân phối bài tập (assignments),...
*   **User**: Học sinh, sinh viên tham gia học tập và tương tác với AI.
*   **Tutor**: Giảng viên, trợ giảng quản lý lớp học, chấm điểm và cấu hình trợ lý ảo cho môn học.

#### Phía Backend (So sánh phương án phân chia)

| Tiêu chí | **Chia theo Role** (admin-be / tutor-be / user-be) | **Chia theo Tính năng** (CMS Service / AI Service) |
| :--- | :--- | :--- |
| **Nguyên tắc phân chia** | Dựa trên **đối tượng sử dụng** | Dựa trên **domain nghiệp vụ** |
| **Độc lập deploy** | ✅ Deploy riêng từng role | ✅ Deploy riêng từng tính năng |
| **Khả năng Scale** | ⚠️ Scale theo role → dễ dư thừa tài nguyên nếu user-be vừa xử lý nghiệp vụ thông thường vừa tải AI nặng | ✅ Scale AI Service độc lập khi nhu cầu tính toán AI/Vector Search tăng cao |
| **Tái sử dụng logic** | ❌ Logic nghiệp vụ bị trùng lặp nhiều giữa các role (ví dụ: xem khóa học có ở cả tutor & user) | ✅ Logic tập trung theo domain, tái sử dụng cao, tránh lặp mã nguồn |
| **Phân quyền (AuthZ)** | ✅ Rõ ràng theo role ngay từ tầng service | ⚠️ Cần xử lý phân quyền tập trung tại API Gateway hoặc phân rã logic trong service |
| **Độ phức tạp** | ⚠️ Nhiều service nhỏ nhưng logic bên trong mỗi service đơn giản hơn | ✅ Ít service hơn, nhưng mỗi service gộp nhiều vai trò nên cần thiết kế module tốt |
| **Phù hợp với team nhỏ** | ❌ Phải duy trì và vận hành 3 repository/pipeline riêng biệt | ✅ Chỉ cần vận hành

### 1.2.3. Layer 3: Tầng dữ liệu (Database & Storage)

Ở mốc triển khai 5 năm, hệ thống được thiết kế theo hướng **tự host trên cụm Kubernetes (On-premise)**. Mỗi loại dữ liệu với đặc tính riêng sẽ được lưu trữ bởi các giải pháp chuyên dụng:

*   **File Storage (Tài liệu, slide, bài nộp)**: Cần dung lượng lớn, hỗ trợ chia sẻ đa node.
*   **PostgreSQL**: Cần tính nhất quán cao, hỗ trợ HA, backup và khôi phục tự động.
*   **Redis**: Cần tốc độ cực cao để lưu cache/session/rate limit tạm thời.
*   **Vector Database**: Cần độ trễ cực thấp và I/O đĩa cực lớn để phục vụ tìm kiếm ngữ nghĩa RAG.

```mermaid
graph TD
    subgraph Clients["Tầng Ứng Dụng (K8s Pods)"]
        CMS["📦 CMS Service"]
        AI["📦 AI Service"]
        Mon["📊 Monitoring Stack"]
    end

    subgraph StorageClasses["Tầng Lưu Trữ (Storage Classes)"]
        classDef classLocal fill:#f9f,stroke:#333,stroke-width:2px;
        classDef classNet fill:#bbf,stroke:#333,stroke-width:2px;

        L_NVMe["💾 local-nvme\n(NVMe SSD cục bộ)"]:::classLocal
        L_HDD["💾 local-hdd-object\n(HDD Enterprise)"]:::classLocal
        L_Rep["💾 longhorn-replicated\n(Replicated Block Network)"]:::classNet
    end

    subgraph Systems["Hệ thống lưu trữ"]
        MinIO["☁️ MinIO Tenant\n(Object - RWX)"]
        PG["🐘 CloudNativePG\n(Postgres HA - RWO)"]
        Redis["⚡ Redis Sentinel\n(Cache/Session)"]
        Qdrant["🔍 Qdrant StatefulSet\n(Vector DB - RWO)"]
        Longhorn["🛡️ Longhorn CSI\n(Workloads phụ)"]
    end

    subgraph Offsite["Lưu trữ ngoại biên"]
        R2["☁️ Cloudflare R2 / S3"]
    end

    CMS -->|Metadata / Transactions| PG
    CMS -->|Tài liệu / Slide / Assets| MinIO
    CMS & AI -->|Cache / Sessions| Redis
    AI -->|ANN Semantic Search| Qdrant
    Mon -->|Metrics / Logs| Longhorn

    PG -.->|PVC| L_NVMe
    Qdrant -.->|PVC| L_NVMe
    MinIO -.->|PVC| L_HDD
    Redis -.->|PVC| L_Rep
    Longhorn -.->|PVC| L_Rep

    %% Backup Flows
    PG -->|Backup / WAL| MinIO
    Qdrant -->|Snapshot| MinIO
    MinIO -->|Sync dữ liệu quan trọng| R2
```

---

#### 1.2.3.1. File Storage (MinIO)

*   **Giải pháp chọn**: **MinIO Operator + Distributed MinIO Tenant** (tương thích S3 API).
*   **Thiết kế lưu trữ**: Backend không ghi file lên ổ đĩa cục bộ mà lưu metadata vào PostgreSQL và đẩy file vật lý sang MinIO.
*   **Lý do chọn & Điểm nổi bật**:
    *   Hỗ trợ truy cập đa node (**ReadWriteMany - RWX**) qua HTTP API, giúp Backend pod scale ngang thoải mái.
    *   Cơ chế **Erasure Coding** phân tán dữ liệu trên nhiều đĩa/máy chủ giúp tự phục hồi khi hỏng đĩa hoặc sập node.
    *   Dễ dàng chuyển đổi sang các dịch vụ S3 Cloud (Cloudflare R2, AWS S3) sau này mà không cần sửa code.
    *   Làm kho lưu trữ tập trung cho các bản backup PostgreSQL và snapshot của Vector Database.
*   **Cấu hình dự kiến**:
    *   *Dung lượng*: 3 – 6 TB raw (slide, bài nộp, logs, backups).
    *   *Mô hình*: Distributed chạy trên 3 node vật lý.
    *   *Cấu hình mỗi node*: 1 – 2 × HDD Enterprise 8 TB (sử dụng StorageClass `local-hdd-object`).

---

#### 1.2.3.2. PostgreSQL (CloudNativePG)

*   **Giải pháp chọn**: **CloudNativePG Operator** (1 Primary + 2 Replicas).
*   **Lý do chọn & Điểm nổi bật**:
    *   Quản lý PostgreSQL chuẩn Cloud-Native: tự động hóa replication, phát hiện và failover node Primary lỗi trong <1 phút.
    *   Tự động backup dữ liệu và WAL (Write-Ahead Logging) trực tiếp lên cụm MinIO nội bộ, hỗ trợ Point-in-Time Recovery (PITR).
    *   *Tại sao không dùng Longhorn làm chính cho PostgreSQL?* PostgreSQL production cần HA và phục hồi ở tầng database (do CloudNativePG quản lý) thay vì chỉ nhân bản vật lý thô ở tầng block storage (Longhorn). Longhorn chỉ dùng cho đĩa phụ của các workload không nhạy cảm độ trễ.
*   **Cấu hình dự kiến**:
    *   *Dung lượng dữ liệu*: 100 – 300 GB.
    *   *Tài nguyên/Pod*: 2 – 4 core CPU, 8 – 16 GB RAM (Limit: 16 – 32 GB RAM).
    *   *Lưu trữ*: PVC 300 – 500 GB cho mỗi instance sử dụng StorageClass `local-nvme` (đĩa NVMe SSD vật lý gắn trực tiếp trên node máy chủ để tối đa hóa IOPS).

---

#### 1.2.3.3. Redis (Sentinel / Cluster)

*   **Giải pháp chọn**: **Redis Sentinel** (1 Master + 2 Replicas + 3 Sentinels) đảm bảo HA ở quy mô vừa phải.
*   **Lý do chọn**: Tốc độ truy xuất mili-giây trên RAM để lưu session, refresh token, rate limit và cache kết quả SQL thường truy vấn, giảm tải tối đa cho PostgreSQL.
*   **Cấu hình dự kiến**:
    *   *Tài nguyên/Pod*: 500m – 1 core CPU, 2 – 4 GB RAM.
    *   *Lưu trữ*: 20 – 50 GB PVC dùng StorageClass `longhorn-replicated` hoặc `local-ssd` (bật AOF/RDB persistence). Dữ liệu chỉ mang tính chất cache/session, dữ liệu gốc luôn nằm ở PostgreSQL.

---

#### 1.2.3.4. Vector Database (Qdrant)

*   **Giải pháp chọn**: **Qdrant StatefulSet** chạy trên **Local PV/NVMe**.
*   **Lý do chọn & Điểm nổi bật**:
    *   Đơn giản và nhẹ hơn rất nhiều so với Milvus (không cần các thành phần phụ trợ phức tạp như message queue, metadata store), phù hợp với đội vận hành nhỏ.
    *   *Tại sao dùng Local PV/NVMe?* Tìm kiếm vector (ANN Search) yêu cầu độ trễ cực thấp và I/O đĩa rất lớn. Đặt Vector DB trên network block storage (như Longhorn) qua mạng LAN 1Gbps sẽ gây nghẽn băng thông nghiêm trọng. Qdrant bắt buộc phải truy xuất trực tiếp ổ đĩa NVMe cục bộ để đạt hiệu năng tối ưu.
*   **Cấu hình dự kiến**:
    *   *Dung lượng index*: 50 – 150 GB (khoảng 5.000.000 vectors).
    *   *Tài nguyên/Pod*: 2 – 4 core CPU, 8 – 16 GB RAM (Limit: 32 – 64 GB RAM để Qdrant load cache index vào RAM).
    *   *Lưu trữ*: PVC 500 GB – 1 TB dùng StorageClass `local-nvme` (Enterprise NVMe SSD gắn trực tiếp).
    *   *Backup*: Snapshot định kỳ và tự động upload lên MinIO.

---

#### 1.2.3.5. Block Storage phụ (Longhorn)

*   **Giải pháp chọn**: **Longhorn CSI**.
*   **Mục đích sử dụng**: Cấp PVC có nhân bản (replication) cho các workload phụ, không nhạy cảm độ trễ I/O (Monitoring stack Prometheus/Grafana, Dev/Staging DBs, Dashboard metadata). Tuyệt đối **không dùng** Longhorn cho đĩa dữ liệu của MinIO, PostgreSQL chính và Qdrant production.
*   **Cấu hình**: Chế độ Replication = 2 hoặc 3 tùy số node vật lý, sử dụng StorageClass `longhorn-replicated`.

---

#### 1.2.3.6. Chiến lược Backup và Khôi phục

Quy trình backup đa tầng tự động đảm bảo khả năng khôi phục nhanh khi xảy ra thảm họa phần cứng:

1.  **PostgreSQL**: CloudNativePG tự động backup hàng ngày + lưu trữ WAL archive liên tục lên cụm MinIO nội bộ.
2.  **Vector DB**: Qdrant snapshot tự động đẩy lên cụm MinIO.
3.  **Offsite Backup**: Các dữ liệu quan trọng nhất (MinIO buckets chứa code/bài nộp của sinh viên, snapshot Qdrant, base backup PostgreSQL) được định kỳ đồng bộ ngoại biên sang **Cloudflare R2** hoặc **S3-compatible Cloud Storage**.
    *   *RPO mục tiêu*: Từ vài phút đến vài giờ tùy loại dữ liệu.
    *   *RTO mục tiêu*: 15 – 60 phút đối với PostgreSQL; 2 – 4 tiếng đối với dữ liệu Object lớn.

---

#### 1.2.3.7. Cấu hình phần cứng Node tổng thể (Cụm 3 Node vật lý)

Cấu hình tối thiểu đề xuất cho **mỗi node máy chủ vật lý** trong cụm Kubernetes để chạy mượt mà toàn bộ Data Layer mốc 5 năm:

*   **CPU**: 16 – 32 core.
*   **RAM**: 128 GB.
*   **Disk hệ điều hành (OS)**: 1 × SSD 512 GB.
*   **Disk dữ liệu nóng (Hot Data)**: 1 × Enterprise NVMe SSD 1.92 TB (sử dụng cho StorageClass `local-nvme` của PostgreSQL và Qdrant).
*   **Disk lưu trữ đối tượng (Object/Archive)**: 1 – 2 × HDD Enterprise 8 TB (sử dụng cho StorageClass `local-hdd-object` của MinIO).
*   **Mạng**: Kết nối mạng nội bộ tối thiểu **10GbE** (khuyến nghị) hoặc 2.5GbE để đồng bộ dữ liệu Longhorn/MinIO nhanh chóng.

---

#### 1.2.3.8. Tóm tắt giải pháp lựa chọn

| Thành phần | Giải pháp đề xuất | Lý do cốt lõi |
| :--- | :--- | :--- |
| **File Storage** | MinIO Operator + Tenant | Chuẩn S3-compatible, chia sẻ đa node dễ dàng (RWX), Erasure Coding tự bảo vệ dữ liệu. |
| **PostgreSQL** | CloudNativePG Operator | Kubernetes-native, tự động hóa HA (failover < 1 phút), tích hợp sẵn backup/WAL archive lên MinIO. |
| **Redis** | Redis Sentinel | Tốc độ RAM cực nhanh, HA đơn giản với Master/Replica/Sentinel, giảm tải tối đa cho DB chính. |
| **Vector DB** | Qdrant trên Local PV/NVMe | Kiến trúc nhẹ hơn Milvus, tối ưu độ trễ tìm kiếm vector (<10ms) nhờ truy xuất đĩa NVMe cục bộ. |
| **Block Storage phụ** | Longhorn CSI | Cấp đĩa ảo có nhân bản nhanh chóng cho các workload phụ (Monitoring, Dev/Staging). |
| **Offsite Backup** | Cloudflare R2 / S3 Cloud | Phòng ngừa thảm họa mất toàn bộ cụm máy chủ của trường, tối ưu chi phí lưu trữ dài hạn. |

---

**Tham chiếu:** MinIO hỗ trợ triển khai trên Kubernetes bằng operator và dùng erasure coding cho distributed storage; CloudNativePG là operator quản lý PostgreSQL HA với primary/standby và automated failover; Qdrant là vector database/vector search engine có API production-ready; Longhorn là distributed block storage cho Kubernetes và hỗ trợ snapshot/backup.

