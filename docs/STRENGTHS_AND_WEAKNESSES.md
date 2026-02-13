# Điểm mạnh và điểm yếu của AI Scientist v2

Phân tích dựa trên cấu trúc code, pipeline và tài liệu README.

---

## Điểm mạnh

### 1. Pipeline end-to-end tự động

- Từ **ý tưởng** (topic/workshop file) → **ideation** → **thí nghiệm** (BFTS 4 stage) → **tổng hợp figure** → **viết bài** (LaTeX/PDF) → **review** (LLM + VLM). Một lệnh `launch_scientist_bfts.py` có thể chạy toàn bộ (trừ ideation tách riêng).
- Giảm can thiệp tay so với v1: không phụ thuộc template bài báo do con người viết sẵn, phù hợp khám phá mở.

### 2. Kiến trúc thí nghiệm có cấu trúc (4 stage)

- **Stage 1 → 2 → 3 → 4** rõ ràng: implementation → baseline tuning → creative research → ablation.
- Mỗi stage có mục tiêu và số iteration cấu hình được; chuyển stage do LLM đánh giá (function calling), có thể tạo sub-stage (e.g. `1_initial_implementation_1_preliminary`).
- Best node của stage trước được truyền vào stage sau, đảm bảo thí nghiệm xây dựng trên kết quả tốt nhất đã có.

### 3. Tìm kiếm cây (tree search) và song song

- **Best-First Tree Search** với nhiều node (code + metric + bug/analysis), có debug (retry với `max_debug_depth`, `debug_prob`).
- **ParallelAgent** chạy nhiều worker, mở rộng cây nhanh hơn; Journal lưu toàn bộ lịch sử node, dễ tổng hợp và chọn best.
- Có visualization cây (`unified_tree_viz.html`) và checkpoint để theo dõi và khôi phục.

### 4. Đa model và đa backend

- Hỗ trợ nhiều LLM: OpenAI (GPT-4o, o1, o3-mini), Anthropic (Claude), Bedrock, Vertex AI, Gemini, Ollama (Qwen, DeepSeek, …).
- Tách backend (OpenAI vs Anthropic) rõ ràng; có thể chọn model khác nhau cho coding, feedback, writeup, citation, review, aggregate plots.
- Linh hoạt chi phí/chất lượng: ví dụ Claude cho experiment, GPT-4o cho citation, o1 cho writeup.

### 5. Công cụ ngoài (tools) và tìm kiếm tài liệu

- **Semantic Scholar** dùng trong ideation (kiểm tra novelty) và writeup (gather citations), có backoff khi gặp lỗi mạng/rate limit.
- Tool **FinalizeIdea** chuẩn hóa output ideation thành JSON có cấu trúc (Name, Title, Hypothesis, Experiments, Risk Factors, …), dễ dùng cho các bước sau.

### 6. Đánh giá đa kênh (LLM + VLM)

- **LLM review:** Đánh giá toàn văn theo form kiểu NeurIPS (Originality, Quality, Clarity, Significance, Soundness, Decision, …).
- **VLM review:** Xem từng figure, đối chiếu caption và phần trích dẫn trong bài → phát hiện lệch caption/figref, gợi ý cải thiện figure.
- Kết hợp giúp đánh giá vừa nội dung vừa hình ảnh.

### 7. Thực thi code an toàn tương đối

- **Interpreter** chạy code trong process riêng (multiprocessing), có **timeout** (mặc định 3600s), bắt stdout/stderr và exception.
- Traceback được làm gọn (bỏ path nội bộ treesearch) trước khi đưa cho LLM feedback, giảm rò rỉ cấu trúc thư mục.

### 8. Theo dõi token và cấu hình

- **Token tracker** ghi lại usage theo từng tương tác, xuất `token_tracker.json` và `token_tracker_interactions.json` → ước lượng chi phí và debug.
- **bfts_config.yaml** tập trung cấu hình: agent (workers, steps, stages), search (debug, num_drafts), exec (timeout), model từng vai trò. Copy config theo từng run vào idea_dir, tránh ghi đè giữa các run.

### 9. Tổng hợp log và báo cáo

- **Log summarization** gộp nhiều node thành JSON (Experiment_description, Significance, List_of_included_plots, Key_numerical_results) và có bước aggregate qua các stage.
- **Journal → report** sinh markdown báo cáo từ journal; **aggregate_plots** dùng LLM để sinh script tổng hợp figure từ .npy và JSON, thống nhất figure cuối cho bài báo.

### 10. Khả năng mở rộng ý tưởng sang lĩnh vực khác

- Chỉ cần tạo file topic .md mới (Title, Keywords, Abstract, …), chạy ideation, rồi dùng file .json ý tưởng với `--load_ideas`. README nói rõ cách chạy cho subject field khác.

---

## Điểm yếu

### 1. Phụ thuộc mô hình và tỷ lệ thành công không ổn định

- README thừa nhận: **success rate phụ thuộc foundation model và độ phức tạp ý tưởng**; có thể không ra PDF hoặc review nếu experiment/writeup thất bại.
- V2 “broad, exploratory” → **tỷ lệ thành công thường thấp hơn v1** khi đã có template tốt; v1 phù hợp hơn khi mục tiêu rõ và nền tảng sẵn có.

### 2. Rủi ro bảo mật và môi trường thực thi

- README cảnh báo: **code do LLM viết được thực thi** → có thể dùng package nguy hiểm, truy cập web, spawn process ngoài ý muốn.
- Khuyến nghị chạy trong **sandbox (e.g. Docker)**; codebase không tích hợp sẵn sandbox (container, restrict network, allowlist package).

### 3. Chi phí và thời gian

- **Experiment:** ~$15–20/run với Claude 3.5 Sonnet; **writeup** thêm ~$5 với model mặc định. Ideation vài dollar.
- Toàn pipeline **vài giờ**; không có tùy chọn “chỉ chạy stage X” hoặc resume rõ ràng từ checkpoint cho toàn bộ launch script (chỉ BFTS có checkpoint nội bộ).

### 4. Phụ thuộc API bên ngoài

- **Semantic Scholar:** Không có key thì bị rate limit hoặc phải bỏ qua citation/novelty; code có xử lý nhưng trải nghiệm kém hơn.
- **OpenAI/Anthropic/Bedrock:** Mất mạng hoặc API lỗi sẽ làm dừng pipeline; có backoff nhưng không có fallback model tự động.

### 5. Yêu cầu phần cứng và môi trường

- Thiết kế cho **Linux + NVIDIA GPU + CUDA + PyTorch**; chạy experiment do LLM đề xuất có thể cần GPU (README nói có thể gặp CUDA OOM, gợi ý đề xuất model nhỏ hơn trong topic).
- Cần **LaTeX** (pdflatex, bibtex), **poppler**, **chktex**; không có script kiểm tra sẵn đủ dependency.

### 6. Độ phức tạp code và bảo trì

- **AgentManager** và **ParallelAgent** rất dài (hàng nghìn dòng), nhiều branch theo stage và sub-stage → khó đọc, khó sửa lỗi.
- Nhiều magic string (tên stage, key task_desc), ít type hint đầy đủ ở một số chỗ → refactor dễ gãy.

### 7. Giới hạn của interpreter

- Timeout cố định; không có cơ chế “chạy từng cell” hoặc giới hạn tài nguyên (CPU/memory) ngoài timeout.
- Không thấy cơ chế cấm import (e.g. `os.system`, `subprocess`) → trong sandbox kém, code LLM có thể gọi lệnh hệ thống.

### 8. Writeup và figure

- **Aggregate plots** phụ thuộc LLM sinh script từ summary + .npy; prompt dài, dễ hallucinate path hoặc số liệu nếu summary thiếu/không nhất quán.
- Writeup retry cố định (`--writeup-retries`); không có cơ chế tự sửa (e.g. parse lỗi LaTeX rồi gửi lại cho LLM sửa).

### 9. Đánh giá chất lượng khoa học

- Review LLM/VLM chỉ là **gợi ý**; không có tích hợp với reviewer con người hay rubric định lượng.
- Không có bước kiểm tra reproducibility (chạy lại code từ journal để so sánh số liệu).

### 10. License và trách nhiệm sử dụng

- **AI Scientist Source Code License** (derivative of Responsible AI License) yêu cầu **disclosure** rõ ràng việc dùng AI trong manuscript.
- Nếu dùng cho bài gửi hội nghị phải tự đảm bảo tuân thủ policy của từng venue.

---

## Tóm tắt

| Khía cạnh | Điểm mạnh | Điểm yếu |
|-----------|-----------|----------|
| **Tự động hóa** | Pipeline end-to-end, 4 stage có cấu trúc | Success rate không ổn định, phụ thuộc model |
| **Kiến trúc** | Tree search, song song, đa model/backend | Code phức tạp, khó bảo trì |
| **Công cụ** | Semantic Scholar, token tracker, config linh hoạt | Phụ thuộc API, thiếu sandbox mặc định |
| **Chất lượng** | LLM + VLM review, summarization log | Chưa có reproducibility check, review chỉ tham khảo |
| **Vận hành** | Một lệnh chạy đủ, đa GPU/backend | Chi phí/thời gian cao, phụ thuộc LaTeX/GPU |

Tài liệu này giúp nắm nhanh **tính năng** (docs/FEATURES.md) và **ưu/nhược điểm** khi triển khai hoặc mở rộng AI Scientist v2.
