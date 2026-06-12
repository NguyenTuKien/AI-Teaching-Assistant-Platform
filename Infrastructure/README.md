# 1. Demo

## 1.1. Mục tiêu Demo

* **Thời gian phát triển:** khoảng 2 tuần.
* **Quy mô thử nghiệm:** khoảng 10–15 người dùng.
* **Mục tiêu:** kiểm tra tính khả thi của giải pháp, demo các luồng chính, thu thập phản hồi từ giảng viên/sinh viên và phát hiện các vấn đề tiềm ẩn trước khi triển khai rộng hơn.

Ở giai đoạn Demo, hệ thống nên ưu tiên:

> **Open-source core + cloud/free tier để triển khai nhanh và tiết kiệm chi phí.**

Nghĩa là phần lõi vẫn dựa trên các công nghệ open-source hoặc có thể thay thế bằng open-source, nhưng tài nguyên chạy thử như database, Redis, object storage, vector database, hosting frontend nên tận dụng các dịch vụ cloud miễn phí.

---

## 1.2. Chi tiết triển khai Demo

| Thành phần      | Mô tả                                                       | Công nghệ đề xuất                                              | Ghi chú tài nguyên                                   |
| --------------- | ----------------------------------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------- |
| Frontend        | Giao diện cho 2 role: Lecturer và Student                   | Vercel / Cloudflare Pages / Netlify                            | Dùng free tier, không cần server riêng               |
| Backend         | Xử lý nghiệp vụ: user, course, project, progress, dashboard | Tự build hoặc tham khảo Moodle, Open edX, Canvas LMS, Sakai    | Chạy bằng Docker trên VPS nhỏ                        |
| Redis / Cache   | Lưu cache, session tạm thời, queue nhẹ                      | Redis Cloud / Upstash Redis                                    | Dùng free tier                                       |
| Database        | Lưu user, lớp học, bài tập, kết quả, progress               | Supabase PostgreSQL                                            | Dùng free tier, giảm công vận hành                   |
| Object Storage  | Lưu slide, giáo trình, file bài tập, kết quả                | Cloudflare R2 / Supabase Storage                               | Dùng free tier, tránh lưu file trực tiếp trên server |
| Vector Database | Lưu embedding phục vụ RAG                                   | Qdrant Cloud / Pinecone / Weaviate Cloud / Supabase pgvector   | Dùng free tier                                       |
| Gateway         | Định tuyến API, rate limit cơ bản                           | Kong Gateway OSS / Nginx / Traefik                             | Chỉ cần nếu backend tách nhiều service               |
| AI Service      | Xử lý AI tutor, RAG workflow, guardrails                    | RAGFlow, LangChain, LlamaIndex, Haystack hoặc service tự build | Không self-host LLM, gọi API ngoài                   |
| Git Repo        | Quản lý mã nguồn bài nộp/project                            | Gitea self-host hoặc GitHub tạm thời                           | Demo có thể chạy Gitea nhỏ                           |
| CI/Autograding  | Chứng minh luồng push code → review/test → feedback         | Mock worker / Gitea Actions / GitHub Actions                   | Chỉ cần demo flow, chưa cần sandbox phức tạp         |
| Monitoring      | Theo dõi log/lỗi cơ bản                                     | Docker logs, Sentry free, provider logs                        | Chưa cần Prometheus/Grafana đầy đủ                   |

---

## 1.3. Server cần cho Demo

Vì Database, Redis, Object Storage, Vector DB và Frontend đều dùng cloud/free tier, server Demo chỉ cần chạy:

* Backend.
* AI/RAG service.
* Gitea nhỏ hoặc Git integration service.
* Gateway nếu cần.
* Worker nhỏ cho xử lý tài liệu hoặc mock CI.

### Cấu hình tối thiểu

| Tài nguyên |      Đề xuất |
| ---------- | -----------: |
| CPU        |       2 vCPU |
| RAM        |       2–4 GB |
| Storage    | 20–40 GB SSD |
| GPU        |    Không cần |

### Cấu hình khuyến nghị

| Tài nguyên |      Đề xuất |
| ---------- | -----------: |
| CPU        |       4 vCPU |
| RAM        |       4–8 GB |
| Storage    | 40–80 GB SSD |
| GPU        |    Không cần |

Với quy mô 10–15 người dùng, cấu hình 4 vCPU, 4 GB RAM, 40 GB SSD là hợp lý hơn so với 2 GB RAM và 10 GB storage, vì Gitea, backend và AI/RAG service chạy chung sẽ khá dễ thiếu RAM/disk nếu có upload tài liệu hoặc repo code.

---

## 1.4. Kết luận Demo

Bản Demo nên triển khai theo hướng:

> **Dùng cloud/free tier để chạy nhanh, nhưng giữ lõi hệ thống theo hướng open-source để sau này có thể self-host.**

Các dịch vụ nên dùng cloud/free tier ở Demo:

* Vercel / Cloudflare Pages cho Frontend.
* Supabase PostgreSQL cho Database.
* Cloudflare R2 hoặc Supabase Storage cho Object Storage.
* Redis Cloud hoặc Upstash Redis cho Cache/Queue.
* Qdrant Cloud, Pinecone, Weaviate Cloud hoặc pgvector cho Vector DB.
* LLM API ngoài cho AI inference.

Các thành phần nhóm nên tự build hoặc kiểm soát:

* Backend nghiệp vụ.
* AI workflow.
* RAG pipeline.
* Progress tracking.
* Git/CI integration.
* Dashboard cho giảng viên.

---

# 2. Production

## 2.1. Mục tiêu Production

* **Thời gian phát triển:** chưa xác định.
* **Quy mô:** triển khai cho nhiều lớp/môn học, nhiều sinh viên và giảng viên.
* **Mục tiêu:** đảm bảo hiệu suất, bảo mật, khả năng mở rộng, kiểm soát dữ liệu và tính sẵn sàng cao.

Ở Production, hệ thống nên đi theo hướng:

> **Open-source core + self-host toàn bộ tài nguyên quan trọng + High Availability.**

Khác với Demo, Production không nên phụ thuộc vào free tier vì dễ gặp giới hạn quota, thay đổi chính sách, khó kiểm soát dữ liệu và khó đảm bảo vận hành dài hạn.

---

## 2.2. Chi tiết triển khai Production

| Thành phần            | Mô tả                                              | Công nghệ đề xuất                                               | Yêu cầu HA                                |
| --------------------- | -------------------------------------------------- | --------------------------------------------------------------- | ----------------------------------------- |
| Frontend              | Giao diện cho Lecturer và Student                  | Next.js/React deploy trên Kubernetes hoặc Nginx                 | Chạy nhiều replica                        |
| Backend               | Xử lý nghiệp vụ chính                              | Service tự build, có thể tham khảo Moodle/Open edX/Canvas/Sakai | Stateless, scale ngang nhiều replica      |
| API Gateway / Ingress | Routing, TLS, auth, rate limit                     | Kong Gateway OSS / Nginx Ingress / Traefik / Envoy Gateway      | Tối thiểu 2 replica                       |
| Database              | Lưu dữ liệu người dùng, bài tập, kết quả, progress | PostgreSQL + CloudNativePG                                      | Primary/replica, backup định kỳ           |
| Redis / Cache         | Cache, session, queue nhẹ                          | Redis Sentinel / Redis Cluster / Valkey                         | HA, có replica                            |
| Object Storage        | Lưu file tài liệu, bài nộp, artifact, backup       | MinIO Distributed                                               | Nhiều node/disk, chịu lỗi                 |
| Vector Database       | Lưu embedding cho RAG                              | Qdrant self-host / Weaviate self-host / pgvector                | Replica/sharding tùy quy mô               |
| Message Queue         | Điều phối job nặng giữa các service                | RabbitMQ / NATS / Kafka                                         | Cluster mode                              |
| AI Service            | AI tutor, guardrails, prompt workflow              | LangChain, LlamaIndex, Haystack, RAGFlow hoặc service tự build  | Tách khỏi backend chính, scale riêng      |
| Embedding Service     | Sinh embedding cho tài liệu                        | sentence-transformers, TEI hoặc API nội bộ                      | Scale theo số job                         |
| Git Repo              | Quản lý mã nguồn bài nộp/project                   | Gitea self-host                                                 | Dữ liệu đặt trên SSD/NVMe, backup định kỳ |
| CI/Autograding        | Chạy test, kiểm tra code, sinh feedback            | Jenkins, Gitea Actions Runner, Woodpecker CI hoặc worker riêng  | Tách node riêng, có queue                 |
| Monitoring & Logging  | Giám sát, log, tracing, alert                      | Prometheus, Grafana, Loki, Tempo, Alertmanager                  | Lưu trữ riêng, retention policy           |
| Backup                | Sao lưu dữ liệu quan trọng                         | pgBackRest, Velero, Restic, MinIO backup/replication            | Backup ngoài cụm                          |

---

## 2.3. Cấu hình Production khuyến nghị

### Mức Production tối thiểu

Phù hợp cho giai đoạn pilot production hoặc triển khai nội bộ quy mô nhỏ.

| Nhóm node       |                              Số node | Vai trò                                        | Cấu hình gợi ý               |
| --------------- | -----------------------------------: | ---------------------------------------------- | ---------------------------- |
| Kubernetes node |                                    3 | Chạy app, database, storage, monitoring cơ bản | 8–16 core, 32–64 GB RAM/node |
| Storage         | Dùng chung trên node hoặc disk riêng | PostgreSQL, MinIO, Gitea, Qdrant               | SSD/NVMe khuyến nghị         |

Cụm 3 node có thể đạt HA cơ bản, nhưng workload vẫn còn trộn nhiều. Không nên xem đây là cấu hình tối ưu cho quy mô lớn.

---

### Mức Production khuyến nghị

Phù hợp hơn nếu muốn triển khai nghiêm túc, có HA và tách workload rõ ràng.

| Nhóm node           | Số node | Vai trò                                           | Cấu hình gợi ý                         |
| ------------------- | ------: | ------------------------------------------------- | -------------------------------------- |
| Control-plane/Infra |       3 | Kubernetes control-plane, ingress, monitoring nhẹ | 4–8 core, 16–32 GB RAM                 |
| App node            |       2 | Frontend, backend API, auth, dashboard            | 8–16 core, 32–64 GB RAM                |
| Data node           |       3 | PostgreSQL HA, Qdrant, Redis/Valkey, MinIO        | 16 core, 64–128 GB RAM, NVMe + SSD/HDD |
| Worker/CI node      |     1–2 | AI/RAG worker, CI runner, sandbox                 | 16–32 core, 64–128 GB RAM              |
| GPU node            |     0–2 | Self-host LLM/embedding nếu cần                   | Tùy model và ngân sách                 |

Với Production, điểm quan trọng không chỉ là tổng CPU/RAM, mà là **tách workload**. Đặc biệt, không nên để CI/autograding hoặc RAG worker chạy chung với database node vì các job này có thể tiêu tốn CPU/RAM đột biến.

---

## 2.4. Kết luận Production

Bản Production nên triển khai theo hướng:

> **Self-host trên Kubernetes, dùng open-source, có HA cho các thành phần quan trọng.**

Các thành phần nên self-host:

* PostgreSQL HA bằng CloudNativePG.
* Redis/Valkey HA.
* MinIO Distributed cho object storage.
* Qdrant/Weaviate/pgvector cho vector database.
* Gitea cho Git repository.
* Jenkins/Gitea Actions Runner/Woodpecker CI cho CI/autograding.
* Prometheus, Grafana, Loki, Tempo cho monitoring/logging.
* Kong/Nginx/Traefik/Envoy Gateway cho gateway/ingress.

Production có thể vẫn dùng LLM API ngoài ở giai đoạn đầu nếu chưa có GPU, nhưng dữ liệu chính như user, course, progress, repository, file bài nộp, vector index và log vận hành nên nằm trong hạ tầng self-host.

---

# 3. Bottleneck

| Bottleneck                                         | Mô tả                                                                                                                                                    | Giải pháp                                                                                                                                                          |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Nhiều sinh viên hỏi AI cùng lúc                    | Khi nhiều sinh viên cùng chat với AI tutor, backend và AI service dễ bị nghẽn, đặc biệt nếu mỗi request đều gọi LLM và RAG.                              | Tách AI service khỏi backend chính, dùng queue cho request nặng, streaming response, rate limit theo user/course, cache câu hỏi phổ biến.                          |
| RAG retrieval nhiều request cùng lúc               | Mỗi câu hỏi cần truy vấn vector database, lấy context, rerank và ghép prompt. Nếu không filter tốt, truy vấn sẽ chậm.                                    | Chia collection theo course/môn học, dùng metadata filter theo bài học/rubric/project, giới hạn top-k, cache context thường dùng.                                  |
| Giảng viên upload tài liệu lớn                     | PDF, slide, giáo trình lớn cần parse, chunk, embedding và indexing. Nếu xử lý trực tiếp trong backend sẽ làm chậm toàn hệ thống.                         | Upload file vào object storage, xử lý async qua queue, tách RAG worker riêng, giới hạn dung lượng file, hiển thị trạng thái xử lý tài liệu.                        |
| Gần deadline, nhiều sinh viên push code và chạy CI | Khi nhiều nhóm cùng push code hoặc tạo pull request, CI/autograding tiêu tốn nhiều CPU/RAM và có thể ảnh hưởng backend/database.                         | Tách CI runner sang node riêng, giới hạn số job chạy đồng thời, đặt timeout cho job, dùng queue cho webhook, lưu artifact/log vào object storage thay vì Git repo. |
| Dashboard truy vấn nhiều learning events           | Dashboard của giảng viên cần thống kê progress, số lần hỏi AI, trạng thái project, kết quả CI. Query trực tiếp trên bảng events lớn sẽ gây tải database. | Tạo summary table/materialized view, batch insert learning events, partition theo course/time, dùng read replica hoặc event warehouse riêng cho analytics.         |
| Vector DB và database bị quá tải                   | Khi số lượng tài liệu, embedding, chat log, progress và event tăng, database và vector database dễ chậm nếu thiếu index/cache/filter.                    | Thiết kế index hợp lý, partition dữ liệu theo course/time, cleanup dữ liệu cũ, cache metadata, dùng NVMe cho data node, scale replica/sharding khi cần.            |

---

# 4. Kết luận ngắn

Bản Demo nên ưu tiên triển khai nhanh bằng các dịch vụ cloud/free tier như Vercel, Supabase, Cloudflare R2, Redis Cloud/Upstash và Qdrant/Pinecone/Weaviate Cloud. Server Demo chỉ cần chạy backend, AI/RAG service, Gitea nhỏ và gateway nếu cần. Với 10–15 người dùng, cấu hình khuyến nghị là khoảng 4 vCPU, 4–8 GB RAM và 40–80 GB SSD.

Bản Production nên chuyển sang self-host, triển khai trên Kubernetes và dùng các công nghệ open-source có hỗ trợ HA như CloudNativePG, MinIO Distributed, Redis/Valkey HA, Qdrant/Weaviate self-host, Gitea, Jenkins/Gitea Runner, Kong/Nginx Ingress và Prometheus/Grafana/Loki. Mục tiêu là kiểm soát dữ liệu, giảm phụ thuộc vendor, dễ mở rộng và đảm bảo vận hành ổn định khi số lượng người dùng tăng.
