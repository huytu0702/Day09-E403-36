# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Trung  
**Vai trò trong nhóm:** Trace Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Tôi phụ trách vai trò **Trace Owner** — chịu trách nhiệm chính trong Sprint 4: chạy pipeline end-to-end với 15 test questions và 10 grading questions, sinh trace files, tính metrics đánh giá, và thực hiện so sánh kiến trúc single-agent (Day 08) vs multi-agent (Day 09).

**Module/file tôi chịu trách nhiệm:**
- File chính: `eval_trace.py` — sửa để chạy được trên môi trường thực tế, hoàn thiện hàm `compare_single_vs_multi()`
- Output: 15 trace files trong `artifacts/traces/`, `artifacts/eval_report.json`, `artifacts/grading_run.jsonl` (10 câu grading)
- Hỗ trợ: bổ sung `timestamp` vào state trong `graph.py`, khởi tạo ChromaDB storage

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Tôi là người chạy end-to-end toàn bộ pipeline mà Supervisor Owner (`graph.py`) và Worker Owner (`workers/`) đã build. Trace files tôi sinh ra là bằng chứng để đánh giá routing có đúng không, workers có trả output theo contract không. Dữ liệu trong `eval_report.json` cung cấp metrics cho Docs Owner điền vào `routing_decisions.md` và `single_vs_multi_comparison.md`.

**Bằng chứng (commit hash, file, v.v.):**
- Commit `25c86e5`: sửa `eval_trace.py` (+55 dòng thay đổi), sinh 15 trace files, tạo `eval_report.json`, cập nhật `graph.py` và `requirements.txt`
- Commit `4a71b1e`: khởi tạo ChromaDB storage (`chroma_db/`), push `grading_run.jsonl` (10 câu grading)

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi quyết định hoàn thiện hàm `compare_single_vs_multi()` bằng cách thay thế 5 placeholder `TODO` bằng phân tích thực tế dựa trên 15 trace files, thay vì chờ có file kết quả Day 08.

**Lý do:**
Template ban đầu có `"latency_delta": "TODO"`, `"accuracy_delta": "TODO"`, `"avg_confidence": 0.0`. Day 08 là kiến trúc monolith không có trace system, nên không thể so sánh bằng số liệu trực tiếp. Tôi quyết định dùng phân tích dựa trên sự khác biệt kiến trúc kết hợp metrics thật từ Day 09: `avg_confidence=0.517`, `avg_latency=3089ms`, routing distribution 53% retrieval / 46% policy, MCP usage 13%, HITL rate 6%.

**Trade-off đã chấp nhận:**
So sánh thiên về định tính cho Day 08 (không có số liệu) nhưng hoàn toàn định lượng cho Day 09. Kết quả `eval_report.json` vẫn đủ để nhóm điền `single_vs_multi_comparison.md` với ít nhất 2 metrics theo yêu cầu.

**Bằng chứng từ trace/code:**

```python
# eval_trace.py — compare_single_vs_multi() sau khi hoàn thiện
"analysis": {
    "routing_visibility": "Day 09 có route_reason cho từng câu → dễ debug hơn.",
    "latency_tradeoff": "Day 09 trung bình 3089ms/câu do nhiều worker calls + LLM.",
    "debuggability": "Day 09: xem trace → kiểm tra route → test worker độc lập (~5 phút).",
    "accuracy_observation": "Day 09 xử lý tốt multi-hop questions (q13, q15).",
    "abstain_quality": "Day 09 abstain đúng q09 (ERR-403-AUTH), confidence=0.30, HITL trigger.",
}
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Hàm `analyze_traces()` crash khi đọc trace files có chứa dữ liệu tiếng Việt.

**Symptom (pipeline làm gì sai?):**
Khi chạy lệnh phân tích trace, pipeline văng lỗi `UnicodeDecodeError` và ngừng hoạt động nửa chừng. Do đó, report tổng không được sinh ra và không tính toán được bất kỳ metric nào.

**Root cause (lỗi nằm ở đâu?):**
Hàm `analyze_traces()` dùng `open(file)` mặc định để đọc các file cấu hình và history từ `artifacts/traces/`. Trên hệ điều hành Windows, mã hóa đọc file mặc định thường là `cp1252`, không tương thích với dữ liệu `utf-8` của Tiếng Việt xuất ra từ câu trả lời của LLM.

**Cách sửa:**
Tôi thêm tường minh tham số `encoding="utf-8"` vào hàm `open()` ở tất cả các vị trí đọc/ghi file trace, cũng như ở hàm `save_eval_report`. Đồng thời thêm cấu hình fix console stdout cho Windows.

**Bằng chứng trước/sau:**
Trước khi sửa: hệ thống văng lỗi `UnicodeDecodeError` tại câu hỏi đầu tiên. Sau khi sửa: đọc và phân tích thành công 15/15 files, tạo `eval_report.json` lấy ra đúng nội dung tiếng Việt và tính toán chính xác (`avg_latency=3089ms`, `avg_confidence=0.517`).

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi đảm bảo pipeline chạy end-to-end 15/15 test questions + 10/10 grading questions thành công. Mỗi trace file có đúng format bắt buộc (`run_id`, `supervisor_route`, `route_reason`, `workers_called`, `confidence`, `latency_ms`, `worker_io_logs`). Eval report cho thấy hệ thống hoạt động: routing phân bổ hợp lý (53% retrieval, 46% policy), câu q09 (ERR-403-AUTH) trigger HITL đúng với confidence=0.30, câu q15 (multi-hop P1 + access) route đúng vào `policy_tool_worker` với `risk_high=true` và gọi MCP `get_ticket_info`.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Phần so sánh Day 08 còn thiên về định tính vì không có baseline số liệu từ Day 08. Confidence trung bình chỉ 0.517 — cho thấy pipeline cần cải thiện thêm ở retrieval hoặc synthesis.

**Nhóm phụ thuộc vào tôi ở đâu?**
Nếu tôi chưa chạy pipeline và sinh trace, nhóm không có `grading_run.jsonl` để nộp bài và không có dữ liệu để điền vào docs templates.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi phụ thuộc vào Supervisor Owner (routing logic trong `graph.py`) và Worker Owner (3 workers chạy đúng contract) để pipeline cho ra trace có nghĩa.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ chạy lại Day 08 single-agent pipeline với cùng 15 câu hỏi để có số liệu định lượng so sánh. Bằng chứng: `eval_report.json` hiện ghi `"avg_confidence": "N/A"` cho Day 08 trong khi Day 09 có `avg_confidence=0.517`, `avg_latency=3089ms`. Ngoài ra, trace câu `gq07` cho confidence chỉ 0.30 khi hỏi về mức phạt tài chính vi phạm SLA P1 — tôi sẽ kiểm tra xem tài liệu `sla_p1_2026.txt` có chứa thông tin này không, và nếu không thì bổ sung vào data để cải thiện coverage.
