# Các tính năng của AI Scientist v2

Tài liệu mô tả các tính năng chính của hệ thống **AI Scientist v2** – hệ thống nghiên cứu khoa học tự động end-to-end dựa trên tìm kiếm cây (tree search) và nhiều agent LLM.

---

## 1. Tổng quan pipeline

Luồng chính gồm **4 giai đoạn**:

1. **Ideation** – Sinh ý tưởng nghiên cứu từ mô tả chủ đề  
2. **Experiments (BFTS)** – Chạy thí nghiệm qua Best-First Tree Search với Agent Manager  
3. **Writeup** – Viết bản thảo bài báo (LaTeX → PDF)  
4. **Review** – Đánh giá bài báo bằng LLM và VLM (hình ảnh, caption, trích dẫn)

---

## 2. Ideation (Sinh ý tưởng)

**File:** `ai_scientist/perform_ideation_temp_free.py`

- **Mục đích:** Từ một file Markdown mô tả chủ đề (Title, Keywords, TL;DR, Abstract), LLM sinh nhiều ý tưởng nghiên cứu có cấu trúc.
- **Công cụ:**
  - **Semantic Scholar** (`ai_scientist/tools/semantic_scholar.py`): Tìm kiếm tài liệu, kiểm tra novelty.
  - **FinalizeIdea**: Định dạng ý tưởng thành JSON (Name, Title, Short Hypothesis, Related Work, Abstract, Experiments, Risk Factors and Limitations).
- **Tham số:** `--max-num-generations`, `--num-reflections` để điều khiển số ý tưởng và số bước tinh chỉnh.
- **Đầu ra:** File JSON chứa danh sách ý tưởng, dùng làm input cho bước experiments.

---

## 3. Experiments – Best-First Tree Search (BFTS)

**File:** `ai_scientist/treesearch/perform_experiments_bfts_with_agentmanager.py`, `agent_manager.py`, `parallel_agent.py`

### 3.1 Agent Manager & 4 giai đoạn thí nghiệm

`AgentManager` điều phối **4 stage** thí nghiệm tuần tự:

| Stage | Tên | Mục tiêu chính |
|-------|-----|-----------------|
| 1 | **initial_implementation** | Triển khai cơ bản, đúng chức năng, có thể dùng code mẫu nếu có |
| 2 | **baseline_tuning** | Tuning hyperparameters (LR, epochs, batch size), thêm 2 dataset HuggingFace |
| 3 | **creative_research** | Thử cải tiến mới, thí nghiệm sáng tạo, tổng cộng 3 dataset HuggingFace |
| 4 | **ablation_studies** | Phân tích từng thành phần, dùng cùng dataset như stage 3 |

Chuyển stage được điều khiển bởi LLM (function calling): đánh giá hoàn thành stage, sinh cấu hình stage tiếp theo.

### 3.2 Parallel Agent & Tree Search

- **ParallelAgent** (`parallel_agent.py`): Nhiều worker chạy song song, mỗi worker mở rộng cây nghiệm (sinh code → chạy → đánh giá).
- **Journal & Node** (`journal.py`): Mỗi nút lưu code, kết quả chạy, metric, phân tích bug, plot. Cây được tổ chức theo parent/children.
- **Interpreter** (`interpreter.py`): Chạy code Python trong process riêng, có timeout (mặc định 3600s), bắt stdout/stderr và exception.
- **Backend LLM** (`backend/backend_openai.py`, `backend_anthropic.py`): Gọi API (OpenAI, Anthropic/Bedrock, …) để sinh code và feedback.

### 3.3 Phản hồi sau khi chạy code

- **LLM feedback:** Đánh giá log/traceback → `is_bug`, `summary` (đề xuất sửa).
- **VLM feedback:** Phân tích plot (hình ảnh) từ kết quả thí nghiệm → `plot_analyses`.
- **Debug:** Cấu hình `max_debug_depth`, `debug_prob` trong `bfts_config.yaml` để quyết định có retry debug khi node lỗi hay không.

### 3.4 Cấu hình chính (`bfts_config.yaml`)

- `agent.num_workers`, `agent.steps`: Số worker song song và số bước mở rộng cây.
- `agent.stages.stage*_max_iters`: Số iteration tối đa cho từng stage.
- `agent.search.num_drafts`: Số cây độc lập (draft) ở stage 1.
- `exec.timeout`: Timeout chạy code (giây).
- Model cho code, feedback, VLM feedback được cấu hình trong `agent.code`, `agent.feedback`, `agent.vlm_feedback`.

### 3.5 Tổng hợp & báo cáo

- **Log summarization** (`log_summarization.py`): Tổng hợp log nhiều node thành mô tả thí nghiệm, significance, plots, key numerical results (JSON).
- **Journal → Report** (`journal2report.py`): Từ journal sinh báo cáo markdown (Introduction, Methods, Results, …).
- **Tree visualization:** File `unified_tree_viz.html` trong `experiments/.../logs/0-run/` để xem cây nghiệm.

---

## 4. Aggregate plots

**File:** `ai_scientist/perform_plotting.py`

- Đầu vào: Thư mục experiment đã chạy (idea_dir), các file JSON summary và `.npy` từ các stage.
- LLM (ví dụ `o3-mini`) sinh **một script Python** tổng hợp toàn bộ figure cuối cùng cho bài báo.
- Script phải: chỉ dùng dữ liệu có sẵn (.npy, JSON), lưu figure vào `figures/`, có try/except từng plot, tối đa ~12 figure, có thể gộp nhiều subplot.
- Đầu ra: Các file figure trong `figures/` dùng cho writeup.

---

## 5. Writeup (Viết bài báo)

**File:** `ai_scientist/perform_icbinb_writeup.py` (4 trang), `ai_scientist/perform_writeup.py` (8 trang)

- **Gather citations:** Gọi Semantic Scholar (`search_for_papers`) nhiều round (`--num_cite_rounds`) để thu thập trích dẫn.
- **Viết nội dung:** LLM (o1-preview, gpt-4o, …) viết nội dung bài báo dựa trên idea, experiment summaries, figures, citations.
- **LaTeX:** Compile `template.tex` (pdflatex + bibtex) để ra PDF.
- **Writeup type:** `--writeup-type normal` (8 trang) hoặc `icbinb` (4 trang).

---

## 6. Review (Đánh giá bài báo)

**File:** `ai_scientist/perform_llm_review.py`, `ai_scientist/perform_vlm_review.py`

- **LLM review:** Đọc nội dung PDF (qua pypdf/pymupdf4llm), đánh giá theo form kiểu NeurIPS: Summary, Strengths, Weaknesses, Originality, Quality, Clarity, Significance, Questions, Limitations, Ethical Concerns, Soundness, Presentation, Contribution, Overall, Confidence, Decision (Accept/Reject).
- **VLM review:** Gửi từng figure trong PDF cho Vision model để đánh giá:
  - Nội dung figure (Img_description, Img_review).
  - Caption có khớp figure không (Caption_review).
  - Phần main text trích dẫn figure (Figrefs_review).
- Đầu ra: `review_text.txt`, `review_img_cap_ref.json` trong thư mục experiment.

---

## 7. Hạ tầng & tiện ích

- **LLM client** (`ai_scientist/llm.py`): Hỗ trợ OpenAI, Anthropic, Bedrock, Vertex AI, Gemini, Ollama (Qwen, DeepSeek, …), với retry (backoff) và theo dõi token.
- **Token tracker** (`ai_scientist/utils/token_tracker.py`): Ghi lại usage token theo từng tương tác, lưu `token_tracker.json`, `token_tracker_interactions.json`.
- **Config:** `bfts_config.yaml` được copy và chỉnh (desc_file, workspace_dir, data_dir, log_dir) theo từng run trong `bfts_utils.edit_bfts_config_file`.
- **Launch script** (`launch_scientist_bfts.py`): Parse args, load idea JSON, (tùy chọn) load code/dataset ref, chạy BFTS → copy experiment_results → aggregate_plots → writeup → review, sau đó cleanup process (psutil).

---

## 8. Các tính năng phụ / tùy chọn

- **Load code:** `--load_code` để đưa file `.py` cùng tên với file ideas vào prompt “Code To Potentially Use”.
- **Dataset ref:** `--add_dataset_ref` để thêm nội dung `hf_dataset_reference.py` vào idea.
- **Skip writeup/review:** `--skip_writeup`, `--skip_review`.
- **Writeup retries:** `--writeup-retries` để thử viết bài nhiều lần nếu lỗi.
- **Đa GPU:** Có thể chỉ định GPU qua biến môi trường (logic trong `get_available_gpus`).

---

## 9. Tóm tắt luồng dữ liệu

```
Topic (.md) → Ideation → Ideas (.json)
                              ↓
Ideas + (optional) Code → BFTS (AgentManager, 4 stages)
                              ↓
Journal, checkpoints, experiment_results → Aggregate plots → figures/
                              ↓
Idea + summaries + figures + citations → Writeup → template.tex → PDF
                              ↓
PDF → LLM review + VLM (figures/captions/refs) → review_text.txt, review_img_cap_ref.json
```

Tất cả tính năng trên đều hướng tới **một pipeline tự động**: từ ý tưởng → thí nghiệm có cấu trúc → tổng hợp số liệu và figure → viết bài → đánh giá bài, với tối thiểu can thiệp tay.
