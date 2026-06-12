# Kiến trúc Demo, Production và Bottleneck cho hệ thống AI hỗ trợ Project-Based Learning

## 0. Định hướng hệ thống

Dự án không nên được định vị là một hệ thống hỏi đáp tài liệu đơn thuần. Hướng chính xác hơn là:

> **AI Teaching Assistant Platform hỗ trợ dạy học theo Project-Based Learning.**

Trong hệ thống này, RAG/document QA chỉ là một module phụ trợ để AI hiểu tài liệu môn học, slide, rubric, project spec và yêu cầu của giảng viên. Sản phẩm chính cần tập trung vào các luồng project-based learning:

1. Giảng viên thiết kế case study, mini-project, rubric, milestone.
2. Sinh viên hoặc nhóm sinh viên nhận project và làm bài trên hệ thống.
3. Sinh viên push code, cập nhật tiến độ, hỏi AI tutor khi gặp khó khăn.
4. AI đưa gợi ý theo kiểu Socratic, không làm hộ toàn bộ bài.
5. Hệ thống tracking progress, learning events, kết quả CI/autograding.
6. Giảng viên xem dashboard để biết nhóm nào đang chậm, lỗi gì phổ biến, sinh viên nào cần hỗ trợ.

Vì vậy, kiến trúc cần đặt **CourseOps / ProjectOps Backend** ở trung tâm, không đặt RAG làm trung tâm.

---

# 1. Demo

## 1.1. Mục tiêu Demo

* **Thời gian phát triển:** khoảng 2 tuần.
* **Quy mô thử nghiệm:** khoảng 10–15 người dùng.
* **Mục tiêu:** kiểm tra tính khả thi của luồng project-based learning, thu thập phản hồi từ giảng viên/sinh viên, và phát hiện các vấn đề trước khi triển khai rộng hơn.

Demo cần chứng minh được luồng tối thiểu:

> Giảng viên tạo môn học/project → AI gợi ý case study/mini-project/rubric → sinh viên nhận project → sinh viên làm bài/push code/cập nhật tiến độ → AI gợi ý theo context → giảng viên xem dashboard tracking.

Ở giai đoạn Demo, hệ thống nên ưu tiên:

> **Open-source core + cloud/free tier để triển khai nhanh và tiết kiệm chi phí.**

Nghĩa là phần lõi vẫn dựa trên công nghệ open-source hoặc có thể thay thế bằng open-source, nhưng tài nguyên triển khai như database, storage, vector DB, Redis, hosting frontend nên dùng cloud free tier để giảm công vận hành.

---

## 1.2. Chi tiết triển khai Demo

| Thành phần                | Vai trò trong Project-Based Learning                                                         | Công nghệ đề xuất                                                       | Ghi chú tài nguyên                                      |
| ------------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------- |
| Frontend                  | Giao diện cho Lecturer và Student                                                            | Vercel / Cloudflare Pages / Netlify                                     | Dùng free tier, không cần server riêng                  |
| CourseOps Backend         | Quản lý môn học, lớp, sinh viên, project, nhóm, milestone, rubric, deadline                  | Tự build là chính; có thể tham khảo Moodle, Open edX, Canvas LMS, Sakai | Chạy bằng Docker trên VPS nhỏ                           |
| Project Workspace         | Quản lý project, task, milestone, trạng thái nhóm                                            | Có thể tham khảo Taiga / OpenProject, hoặc tự build module đơn giản     | Demo chỉ cần chức năng cơ bản                           |
| AI Project Assistant      | Hỗ trợ giảng viên tạo case study, mini-project, rubric; hỗ trợ sinh viên bằng gợi ý Socratic | LangChain / LlamaIndex / Haystack / RAGFlow hoặc service tự build       | Gọi LLM API ngoài, không cần GPU                        |
| RAG / Document Context    | Đọc slide, giáo trình, rubric, project spec để cung cấp context cho AI                       | Qdrant Cloud / Pinecone / Weaviate Cloud / Supabase pgvector            | Chỉ là module hỗ trợ AI, không phải trung tâm sản phẩm  |
| Database                  | Lưu user, course, project, group, milestone, rubric, progress, result                        | Supabase PostgreSQL                                                     | Dùng free tier, giảm công vận hành                      |
| Object Storage            | Lưu slide, giáo trình, project spec, bài nộp, artifact                                       | Cloudflare R2 / Supabase Storage                                        | Dùng free tier, tránh lưu file trực tiếp trên server    |
| Redis / Cache / Queue nhẹ | Cache, session, job xử lý tài liệu, job AI nhẹ                                               | Redis Cloud / Upstash Redis                                             | Dùng free tier                                          |
| Git Repository            | Quản lý source code bài project của sinh viên                                                | Gitea self-host hoặc GitHub tạm thời                                    | Demo có thể chạy Gitea nhỏ                              |
| CI / Autograding          | Chạy test đơn giản, kiểm tra bài nộp, sinh feedback                                          | Mock worker / Gitea Actions / GitHub Actions                            | Demo chỉ cần chứng minh flow, chưa cần sandbox phức tạp |
| Gateway                   | Định tuyến API, auth middleware, rate limit cơ bản                                           | Kong Gateway OSS / Nginx / Traefik                                      | Chỉ cần nếu backend tách nhiều service                  |
| Monitoring                | Theo dõi log/lỗi cơ bản                                                                      | Docker logs, Sentry free, provider logs                                 | Chưa cần Prometheus/Grafana đầy đủ                      |

---

## 1.3. Server cần cho Demo

Vì Frontend, Database, Redis, Object Storage và Vector DB đều dùng cloud/free tier, server Demo chỉ cần chạy:

* CourseOps Backend.
* AI Project Assistant service.
* RAG/document worker nhỏ.
* Gitea nhỏ hoặc Git integration service.
* Mock CI/autograding worker.
* Gateway nếu cần.

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

Với Demo 10–15 người dùng, cấu hình khuyến nghị nên là **4 vCPU, 4–8 GB RAM, 40–80 GB SSD**. Lý do là backend, AI service, Gitea và worker nếu chạy chung trên một server có thể nhanh chóng thiếu RAM hoặc disk khi có repo code, file upload và log.

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

* CourseOps Backend.
* Project Workspace.
* AI Project Assistant.
* RAG/document pipeline.
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

Production không nên phụ thuộc vào free tier vì dễ gặp giới hạn quota, thay đổi chính sách, khó kiểm soát dữ liệu sinh viên và khó đảm bảo vận hành dài hạn.

---

## 2.2. Chi tiết triển khai Production

| Thành phần                | Vai trò trong Project-Based Learning                                                  | Công nghệ đề xuất                                               | Yêu cầu HA                             |
| ------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------- |
| Frontend                  | Giao diện Lecturer, Student và Admin                                                         | Next.js/React deploy trên Kubernetes hoặc Nginx                 | Chạy nhiều replica                     |
| API Gateway / Ingress     | Routing, TLS, auth, rate limit                                                        | Kong Gateway OSS / Nginx Ingress / Traefik / Envoy Gateway      | Tối thiểu 2 replica                    |
| CourseOps Backend         | Quản lý course, class, student, lecturer, project, group, milestone, rubric, deadline | Service tự build; có thể tham khảo Moodle/Open edX/Canvas/Sakai | Stateless, scale ngang nhiều replica   |
| Project Workspace Service | Quản lý task, milestone, trạng thái nhóm, submission, peer activity                   | Tự build hoặc tham khảo Taiga/OpenProject                       | Scale ngang, lưu state trong database  |
| AI Project Assistant      | Hỗ trợ giảng viên tạo project/case study/rubric; hỗ trợ sinh viên bằng gợi ý Socratic | LangChain, LlamaIndex, Haystack, RAGFlow hoặc service tự build  | Tách khỏi backend chính, scale riêng   |
| RAG / Document Pipeline   | Parse slide, giáo trình, project spec, rubric để tạo context cho AI                   | Parser worker, chunker, embedding worker, indexing worker       | Xử lý async qua queue                  |
| Database                  | Lưu user, course, project, group, task, milestone, rubric, progress, learning events  | PostgreSQL + CloudNativePG                                      | Primary/replica, backup định kỳ        |
| Vector Database           | Lưu embedding cho tài liệu, project spec, rubric                                      | Qdrant self-host / Weaviate self-host / pgvector                | Replica/sharding tùy quy mô            |
| Object Storage            | Lưu slide, giáo trình, bài nộp, CI artifact, backup                                   | MinIO Distributed                                               | Nhiều node/disk, chịu lỗi              |
| Redis / Cache             | Cache, session, rate limit, dữ liệu tạm                                               | Redis Sentinel / Redis Cluster / Valkey                         | HA, có replica                         |
| Message Queue             | Điều phối job nặng: parse tài liệu, embedding, CI, AI feedback                        | RabbitMQ / NATS / Kafka                                         | Cluster mode                           |
| Git Repository            | Quản lý source code project của sinh viên                                             | Gitea self-host                                                 | Dữ liệu trên SSD/NVMe, backup định kỳ  |
| CI / Autograding          | Chạy test, kiểm tra code, phân tích lỗi, sinh feedback                                | Jenkins, Gitea Actions Runner, Woodpecker CI hoặc worker riêng  | Tách node riêng, có queue              |
| Dashboard / Analytics     | Thống kê progress, nhóm chậm tiến độ, lỗi phổ biến, kết quả CI                        | Service tự build + summary table/materialized view              | Tách query analytics khỏi DB giao dịch |
| Monitoring & Logging      | Giám sát hệ thống, log, tracing, alert                                                | Prometheus, Grafana, Loki, Tempo, Alertmanager                  | Retention policy, storage riêng        |
| Backup                    | Sao lưu database, repo, object storage, config                                        | pgBackRest, Velero, Restic, MinIO replication                   | Backup ngoài cụm                       |

---

## 2.3. Cấu hình Production khuyến nghị

### Mức Production tối thiểu

Phù hợp cho pilot production hoặc triển khai nội bộ quy mô nhỏ.

| Nhóm node       |                             Số node | Vai trò                                        | Cấu hình gợi ý               |
| --------------- | ----------------------------------: | ---------------------------------------------- | ---------------------------- |
| Kubernetes node |                                   3 | Chạy app, database, storage, monitoring cơ bản | 8–16 core, 32–64 GB RAM/node |
| Storage         | Dùng disk riêng hoặc local SSD/NVMe | PostgreSQL, MinIO, Gitea, Qdrant               | SSD/NVMe khuyến nghị         |

Cụm 3 node có thể đạt HA cơ bản, nhưng workload còn trộn nhiều. Không nên để CI/autograding hoặc RAG worker chạy quá nặng trên cùng node với database.

---

### Mức Production khuyến nghị

Phù hợp hơn nếu muốn triển khai nghiêm túc, có HA và tách workload rõ ràng.

| Nhóm node           | Số node | Vai trò                                                       | Cấu hình gợi ý                         |
| ------------------- | ------: | ------------------------------------------------------------- | -------------------------------------- |
| Control-plane/Infra |       3 | Kubernetes control-plane, ingress, monitoring nhẹ             | 4–8 core, 16–32 GB RAM                 |
| App node            |       2 | Frontend, CourseOps Backend, Project Workspace, Dashboard API | 8–16 core, 32–64 GB RAM                |
| Data node           |       3 | PostgreSQL HA, Qdrant, Redis/Valkey, MinIO                    | 16 core, 64–128 GB RAM, NVMe + SSD/HDD |
| Worker/CI node      |     1–2 | RAG worker, AI worker, CI runner, sandbox                     | 16–32 core, 64–128 GB RAM              |
| GPU node            |     0–2 | Self-host LLM/embedding nếu cần                               | Tùy model và ngân sách                 |

Với Production, điểm quan trọng là **tách workload theo bản chất tải**:

* App/API cần scale ngang.
* Database, vector DB, object storage cần ổn định và có backup.
* AI/RAG worker cần xử lý async.
* CI/autograding cần node riêng để không ảnh hưởng hệ thống chính.
* Dashboard analytics nên tránh query trực tiếp quá nặng vào database giao dịch.

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

Production có thể vẫn dùng LLM API ngoài ở giai đoạn đầu nếu chưa có GPU. Tuy nhiên, dữ liệu chính như user, course, project, group, progress, repository, file bài nộp, vector index và log vận hành nên nằm trong hạ tầng self-host.

---

# 3. Bottleneck

| Bottleneck                                             | Mô tả                                                                                                                                                                             | Giải pháp                                                                                                                                                                              |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Nhiều sinh viên/nhóm hỏi AI cùng lúc                   | Trong project-based learning, sinh viên thường hỏi AI khi gặp lỗi, cần giải thích yêu cầu, hoặc xin gợi ý hướng làm. Nếu nhiều nhóm hỏi cùng lúc, AI service và LLM API dễ nghẽn. | Tách AI Project Assistant khỏi backend chính, dùng queue cho request nặng, streaming response, rate limit theo user/course/project, cache câu hỏi phổ biến, phân tầng model rẻ/mạnh.   |
| RAG retrieval theo project/rubric bị chậm              | AI cần lấy context từ slide, giáo trình, project spec, rubric, milestone và tiến độ nhóm. Nếu truy vấn vector quá rộng, phản hồi sẽ chậm và dễ sai context.                       | Chia collection theo course/project, dùng metadata filter theo lesson/rubric/milestone/group, giới hạn top-k, cache context thường dùng, rerank trên tập nhỏ.                          |
| Giảng viên upload nhiều tài liệu/project spec lớn      | Slide, giáo trình, project spec, rubric cần parse, chunk, embedding và indexing. Nếu xử lý trực tiếp trong backend sẽ làm chậm luồng tạo project.                                 | Upload file vào object storage, xử lý async qua queue, tách RAG/document worker riêng, giới hạn dung lượng file, hiển thị trạng thái “đang xử lý tài liệu”.                            |
| Gần deadline, nhiều nhóm push code và chạy CI          | Khi nhiều nhóm cùng push code hoặc tạo pull request, CI/autograding tiêu tốn CPU/RAM lớn và có thể ảnh hưởng backend, database, Git repo.                                         | Tách CI runner sang node riêng, giới hạn số job chạy đồng thời, đặt timeout cho job, queue hóa webhook, lưu artifact/log vào object storage thay vì Git repo.                          |
| Dashboard tracking progress truy vấn quá nặng          | Giảng viên cần xem tiến độ nhóm, milestone, số lần hỏi AI, kết quả CI, learning events. Nếu query trực tiếp trên bảng events lớn sẽ gây tải database.                             | Tạo summary table/materialized view, batch insert learning events, partition theo course/time, dùng read replica hoặc event warehouse riêng cho analytics.                             |
| Database/Git/Object Storage tăng nhanh theo số project | Mỗi môn học có nhiều nhóm, mỗi nhóm có repo, artifact, log CI, file nộp bài và lịch sử progress. Dữ liệu tăng nhanh hơn hệ thống hỏi đáp tài liệu thông thường.                   | Đặt quota theo course/project, bắt buộc `.gitignore`, giới hạn file size trong Git, lưu artifact vào MinIO, lifecycle policy cho object storage, backup định kỳ và cleanup dữ liệu cũ. |

---

# 4. Kết luận ngắn

Bản Demo nên tập trung chứng minh luồng project-based learning tối thiểu: giảng viên tạo project/case study/rubric, sinh viên làm project theo nhóm, AI hỗ trợ bằng gợi ý Socratic, hệ thống tracking progress và giảng viên xem dashboard. Demo nên dùng cloud/free tier như Vercel, Supabase, Cloudflare R2, Redis Cloud/Upstash, Qdrant/Pinecone/Weaviate Cloud để triển khai nhanh. Server Demo chỉ cần chạy CourseOps Backend, AI Project Assistant, RAG worker nhỏ, Gitea hoặc Git integration và mock CI. Với 10–15 người dùng, cấu hình khuyến nghị là khoảng 4 vCPU, 4–8 GB RAM và 40–80 GB SSD.

Bản Production nên self-host trên Kubernetes, dùng open-source và có HA. Trọng tâm Production là kiểm soát dữ liệu project, repo code, progress, bài nộp, rubric, vector index và log vận hành. Các thành phần quan trọng như PostgreSQL, MinIO, Qdrant/Weaviate, Redis/Valkey, Gitea, Jenkins/Gitea Runner, Kong/Nginx Ingress và Prometheus/Grafana/Loki nên được triển khai trong hạ tầng tự quản để đảm bảo bảo mật, khả năng mở rộng và vận hành dài hạn.
