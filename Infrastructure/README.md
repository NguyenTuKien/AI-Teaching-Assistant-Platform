# Tóm tắt kiến trúc Demo, Prod và Bottleneck

## 1. Bản Demo

Bản Demo ưu tiên triển khai nhanh, tiết kiệm chi phí và chứng minh được các luồng chính của hệ thống. Hướng phù hợp là **hybrid/free-first**, tức là tận dụng tài nguyên free/managed service có sẵn, chỉ tự chạy các thành phần cần custom logic.

### 1.1. Thành phần và công nghệ Demo

| Thành phần     | Công nghệ đề xuất                                             | Ghi chú                                                     |
| -------------- | ------------------------------------------------------------- | ----------------------------------------------------------- |
| Frontend       | Vercel, Cloudflare Pages, Netlify hoặc Nginx container        | Gồm 3 site: Admin, User/Sinh viên, Tutor                    |
| Backend/API    | 1 backend service trên VPS, Render/Railway/Fly.io hoặc Docker | Xử lý nghiệp vụ chính: course, project, progress, dashboard |
| Database       | Supabase Postgres hoặc PostgreSQL local                       | Demo nên ưu tiên Supabase để giảm vận hành                  |
| Object Storage | Cloudflare R2, Supabase Storage hoặc MinIO local              | Lưu slide, giáo trình, project spec, bài nộp                |
| Vector DB      | Qdrant Cloud, Supabase pgvector hoặc Qdrant local             | Lưu embedding phục vụ RAG                                   |
| Cache/Queue    | Upstash Redis hoặc Redis local                                | Cache, queue nhẹ cho RAG/job                                |
| AI/LLM         | OpenAI/Gemini/Claude API hoặc model API ngoài                 | Không cần GPU nếu gọi API ngoài                             |
| RAG Worker     | Service tự build                                              | Parse tài liệu, chunking, embedding, indexing               |
| Git Repository | Gitea nhỏ hoặc GitHub/Gitea cloud tạm thời                    | Demo có thể dùng tạm, Prod ưu tiên Gitea self-host          |
| CI/Autograding | Mock worker, Jenkins nhỏ, Gitea Actions hoặc GitHub Actions   | Chỉ cần chứng minh luồng nộp bài → nhận feedback            |
| Monitoring     | Provider logs, Docker logs, Sentry free tier, Portainer       | Chưa cần full monitoring stack                              |

### 1.2. Yêu cầu phần cứng Demo

Nếu dùng managed service như Supabase, R2, Qdrant Cloud, Upstash:

| Hạng mục          | Đề xuất                                 |
| ----------------- | --------------------------------------- |
| Số server tự quản | 0–1 server                              |
| CPU               | 2–4 vCPU                                |
| RAM               | 4–8 GB                                  |
| Disk              | 50–100 GB SSD                           |
| Network           | 1 Gbps hoặc cloud network mặc định      |
| GPU               | Không cần                               |
| Phù hợp           | 30–50 user đồng thời, 1–2 môn học pilot |

Nếu muốn chạy toàn bộ bằng Docker Compose local:

| Hạng mục |  Tối thiểu | Khuyến nghị |
| -------- | ---------: | ----------: |
| CPU      |     4 vCPU |      8 vCPU |
| RAM      |       8 GB |       16 GB |
| Disk     | 200 GB SSD |  500 GB SSD |
| GPU      |  Không cần |   Không cần |

### 1.3. Kết luận Demo

Bản Demo nên ưu tiên **cloud-first** để giảm công vận hành. Nhóm chỉ cần tập trung build frontend, backend, AI workflow và RAG pipeline. Tuy nhiên, vẫn nên chuẩn bị **Docker Compose local fallback** để có thể chạy offline hoặc thay thế khi free tier gặp giới hạn.

---

## 2. Bản Prod

Bản Prod hướng tới triển khai ổn định trên Kubernetes, có khả năng mở rộng, kiểm soát dữ liệu nội bộ và tách riêng các workload nặng như AI/RAG, database, Git và CI/autograding.

### 2.1. Thành phần và công nghệ Prod

| Thành phần            | Công nghệ đề xuất                                                  | Ghi chú                                         |
| --------------------- | ------------------------------------------------------------------ | ----------------------------------------------- |
| Frontend              | 3 site riêng: Admin, User/Sinh viên, Tutor                         | Deploy độc lập, scale riêng                     |
| Backend/API           | Modular services hoặc microservices trên Kubernetes                | Stateless, scale ngang bằng replica/HPA         |
| API Gateway / Ingress | Cloudflare, Cloudflare Tunnel, Ingress Controller hoặc Gateway API | TLS, routing, rate limit                        |
| AI/Tutor Service      | Tutor orchestration, prompt workflow, guardrails                   | Tách khỏi API chính                             |
| RAG Pipeline          | Parser, chunker, embedding worker, indexing worker                 | Xử lý async qua queue                           |
| Database              | PostgreSQL HA bằng CloudNativePG                                   | Dữ liệu user, course, project, progress, rubric |
| Vector DB             | Qdrant self-host                                                   | Nên dùng NVMe cho tốc độ truy vấn               |
| Object Storage        | MinIO distributed                                                  | Lưu tài liệu, bài nộp, artifact, backup         |
| Cache/Queue           | Redis Sentinel/Redis Cluster hoặc queue chuyên dụng                | Cache, session, job queue                       |
| Git Repository        | Gitea self-host                                                    | Quản lý repo, PR, issue, webhook                |
| CI/Autograding        | Jenkins độc lập, Gitea Actions Runner hoặc worker riêng            | Chạy trên node riêng, sandbox cách ly           |
| Monitoring            | Prometheus, Grafana, Loki, Tempo, Alertmanager                     | Metrics, logs, tracing, alert                   |
| Storage               | Local NVMe, MinIO, Longhorn cho workload phụ                       | Data nóng nên ưu tiên NVMe                      |

### 2.2. Các mức cluster Prod

| Mức triển khai   |  Số node | Phù hợp                                             |
| ---------------- | -------: | --------------------------------------------------- |
| Prod tối thiểu   |   3 node | Pilot production, tải vừa, workload còn trộn nhiều  |
| Prod nhỏ         |   5 node | Một vài lớp/môn học, đã tách app/data/worker cơ bản |
| Prod khuyến nghị | 7–9 node | Mục tiêu khoảng 1000 user đồng thời                 |
| Prod mở rộng     | 10+ node | Nhiều môn/khoa, tải lớn dài hạn                     |

### 2.3. Cấu hình Prod khuyến nghị

| Nhóm node           | Số node | Vai trò                                                    | Cấu hình gợi ý                              |
| ------------------- | ------: | ---------------------------------------------------------- | ------------------------------------------- |
| Control-plane/Infra |       3 | Kubernetes control-plane, ingress, system service nhẹ      | 4–8 core, 16–32 GB RAM                      |
| App node            |       2 | Frontend, backend API, auth, course service, dashboard API | 8–16 core, 32–64 GB RAM                     |
| Data node           |       3 | PostgreSQL HA, Qdrant, Redis, MinIO                        | 16 core, 64–128 GB RAM, NVMe + HDD/SSD data |
| Worker/CI node      |     1–2 | AI/RAG worker, Jenkins/Gitea runner, sandbox               | 16–32 core, 64–128 GB RAM                   |

### 2.4. Storage Prod

| Loại dữ liệu          | Nên dùng                                      | Lý do                                  |
| --------------------- | --------------------------------------------- | -------------------------------------- |
| PostgreSQL            | Local NVMe + CloudNativePG backup/replication | Cần IOPS cao                           |
| Qdrant/vector DB      | Local NVMe                                    | Truy vấn vector và indexing cần tốc độ |
| File/tài liệu/bài nộp | MinIO distributed                             | Dễ mở rộng, phù hợp object lớn         |
| Gitea repositories    | SSD/NVMe riêng                                | Tránh chung với CI artifact            |
| CI artifact/log       | MinIO hoặc object storage riêng               | Không làm phình Git repo               |
| Monitoring data       | Longhorn hoặc storage bền vững riêng          | Có retention policy                    |
| Backup                | Offsite/S3-compatible storage                 | Phục hồi khi có sự cố                  |

### 2.5. Kết luận Prod

Cụm 3 node chỉ nên xem là mức tối thiểu hoặc pilot production. Nếu mục tiêu là khoảng 1000 user đồng thời, nên hướng tới 7–9 node trở lên để tách rõ app, data, AI/RAG worker, CI/autograding và monitoring. Đặc biệt, không nên để CI/autograding hoặc RAG worker chạy chung với database node.

---

## 3. Bottleneck, lý do và giải pháp

| Bottleneck                      | Lý do                                                                 | Giải pháp                                                                |
| ------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| AI inference / LLM response     | LLM phản hồi chậm, nhiều sinh viên hỏi cùng lúc                       | Streaming response, cache câu hỏi phổ biến, rate limit, fallback model   |
| LLM API quota / chi phí         | Nếu dùng LLM ngoài có thể bị giới hạn quota hoặc tăng chi phí         | Giới hạn lượt hỏi, phân tầng model rẻ/mạnh, semantic cache               |
| RAG retrieval                   | Truy vấn vector nhiều và rộng làm chậm phản hồi                       | Tách collection theo course, filter theo lesson/rubric, rerank top-k nhỏ |
| Upload tài liệu lớn             | PDF/slide lớn làm nghẽn backend nếu xử lý trực tiếp                   | Upload vào object storage, xử lý async qua queue                         |
| Parse/OCR/chunking              | CPU-bound, mất thời gian                                              | Tách RAG worker, scale worker theo backlog                               |
| Embedding/indexing              | Gọi embedding API chậm hoặc tốn chi phí, indexing nặng                | Batch embedding, cache embedding, retry có kiểm soát                     |
| Vector DB phình to              | Nhiều môn học và tài liệu làm index lớn                               | Chia collection theo course, metadata filter, cleanup dữ liệu cũ         |
| PostgreSQL quá tải              | Nhiều chat logs, learning events, progress tracking                   | Batch insert, partition theo course/time, index hợp lý                   |
| Dashboard analytics nặng        | Query thống kê đè lên database giao dịch                              | Summary table, materialized view, read replica                           |
| CI/autograding nghẽn            | Gần deadline có nhiều PR/push, test tốn CPU/RAM                       | Queue hóa webhook, giới hạn worker đồng thời, timeout job                |
| CI ảnh hưởng hệ thống chính     | Nếu chạy chung với API/DB sẽ làm chậm toàn hệ thống                   | Tách CI node, resource quota, sandbox riêng                              |
| Git repository phình dung lượng | Sinh viên commit artifact, `.jar`, `target/`, dataset                 | Bắt buộc `.gitignore`, giới hạn file size, artifact lưu object storage   |
| Object storage đầy              | File, bài nộp, artifact, backup tăng nhanh                            | Quota theo course, lifecycle policy, disk monitoring                     |
| Redis/Queue nghẽn               | Nhiều job AI/RAG/CI cùng lúc làm backlog dài                          | Tách queue theo loại job, priority queue, concurrency limit              |
| WebSocket/session lỗi khi scale | Backend nhiều replica nhưng session lưu trong RAM                     | Redis session store, sticky session hoặc event bus                       |
| Monitoring phình dữ liệu        | Logs/metrics/traces tăng nhanh                                        | Retention policy, sampling trace, giới hạn log level                     |
| Backup/restore lỗi              | HA không thay thế backup, vẫn có rủi ro xóa nhầm/hỏng dữ liệu         | Backup định kỳ, restore drill, backup ngoài cụm                          |
| Sandbox security                | CI chạy code sinh viên có rủi ro chiếm tài nguyên hoặc escape sandbox | Node riêng, network policy, seccomp/AppArmor, timeout, quota             |
| Secret bị lộ                    | API key, DB password, LLM key nếu commit nhầm sẽ nguy hiểm            | Secret manager, không commit `.env`, rotate key định kỳ                  |
| Vendor dependency ở Demo        | Free tier có thể đổi quota/chính sách                                 | Có Docker Compose local fallback                                         |

## 4. Kết luận ngắn

Bản Demo nên dùng hướng **hybrid/free-first**: Supabase cho database, R2/Supabase Storage cho object storage, Qdrant Cloud/pgvector cho vector DB, Upstash Redis cho cache/queue, LLM API ngoài cho AI. Phần tự build gồm frontend, backend, AI workflow, RAG worker và autograding đơn giản. Cấu hình server nhỏ 2–4 vCPU, 4–8 GB RAM là đủ nếu dùng managed service.

Bản Prod nên dùng **Kubernetes self-host**, tách app, data, AI/RAG, CI/autograding và monitoring. Với mục tiêu khoảng 1000 user đồng thời, nên hướng tới 7–9 node trở lên. Các bottleneck chính nằm ở AI/RAG, upload tài liệu, vector DB, PostgreSQL event tracking, CI/autograding, object storage và backup/restore.
