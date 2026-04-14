# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Phạm V. (phamv)  
**Vai trò trong nhóm:** Worker Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Tôi phụ trách triển khai **3 workers chính** của hệ Supervisor–Worker và cập nhật **worker contracts** để đảm bảo I/O rõ ràng, test độc lập được. Cụ thể, tôi làm ở các file `workers/retrieval.py`, `workers/policy_tool.py`, `workers/synthesis.py` và đồng bộ mô tả trong `contracts/worker_contracts.yaml`. Retrieval worker chịu trách nhiệm lấy evidence (chunks + sources), policy/tool worker kiểm tra ngoại lệ và ghi lại tool calls, synthesis worker tổng hợp câu trả lời grounded kèm citation và confidence.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Workers của tôi là “khối xử lý” để Supervisor (do Supervisor Owner) gọi theo routing quyết định. Khi workers trả output theo contract (retrieved_chunks/policy_result/final_answer + worker_io_logs), phần trace & docs có thể đọc trace và đánh giá lỗi theo “Routing Error Tree”.

**Bằng chứng (commit hash, file, v.v.):**
Commit `1b90c38` kết nối `graph.py` với workers thật và cập nhật contract/status.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi chọn thiết kế workers theo hướng **fail-soft + abstain**, nghĩa là nếu thiếu dependency (Chroma/LLM) hoặc evidence không đủ thì worker **không bịa** mà trả kết quả rỗng/abstain, đồng thời ghi log I/O để trace được.

**Lý do:**
Trong lab này, môi trường mỗi máy có thể thiếu `chromadb` hoặc chưa set API key. Nếu retrieval hoặc synthesis “crash hard”, cả graph sẽ vỡ và không có trace để debug. Fail-soft giúp pipeline vẫn chạy end-to-end, lưu trace và thể hiện rõ “điểm nghẽn” nằm ở retrieval hay synthesis.

**Trade-off đã chấp nhận:**
Khi fail-soft, câu trả lời có thể bị “low value” (confidence thấp, message báo thiếu thông tin). Tôi chấp nhận điều này vì mục tiêu lab là **observability + modularity** hơn là tối ưu chất lượng answer khi môi trường chưa đủ.

**Bằng chứng từ trace/code:**
Retrieval trả rỗng khi query Chroma thất bại (không bịa chunk), và synthesis trả message lỗi/abstain với confidence thấp:

```python
# workers/retrieval.py
except Exception as e:
    print(f"⚠️  ChromaDB query failed: {e}")
    return []
```

```python
# workers/synthesis.py
if not chunks:
    return 0.1  # Không có evidence → low confidence
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Contract và state không khớp naming/field khiến trace thiếu trường cần thiết để debug (đặc biệt `worker_io_logs`).

**Symptom (pipeline làm gì sai?):**
Khi chạy graph, workers có ghi `state.setdefault("worker_io_logs", []).append(worker_io)`, nhưng nếu state ban đầu không có field này thì trace format dễ thiếu nhất quán (một số node/flow không có log I/O từ đầu). Việc thiếu `worker_io_logs` làm Sprint 4 khó đọc “input/output từng worker” theo yêu cầu contract.

**Root cause:**
Trong `graph.py`, `AgentState` và `make_initial_state()` ban đầu chưa khai báo/khởi tạo `worker_io_logs`, trong khi contract lại yêu cầu và workers đã dùng field này.

**Cách sửa:**
Tôi bổ sung `worker_io_logs: list` vào `AgentState` và khởi tạo `"worker_io_logs": []` trong `make_initial_state()` để đảm bảo mọi run đều có mảng log thống nhất.

**Bằng chứng trước/sau:**
Sau sửa, mọi trace JSON luôn có key `worker_io_logs` ngay từ đầu, và workers append log I/O đúng contract (xem các file trace mới trong `artifacts/traces/`).

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi làm rõ “ranh giới trách nhiệm” giữa các workers bằng contract và fail-soft behavior, giúp hệ thống chạy được ngay cả khi môi trường chưa đủ. Điều này làm trace hữu ích hơn: nhìn vào `retrieved_chunks`, `policy_result`, `worker_io_logs` là biết lỗi nằm ở đâu.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Policy analysis hiện còn rule-based đơn giản, chưa truly grounded bằng cách trích nguyên văn rule từ tài liệu, và synthesis vẫn phụ thuộc LLM key để ra câu trả lời chất lượng.

**Nhóm phụ thuộc vào tôi ở đâu?**
Nếu tôi không hoàn tất I/O contract + workers, Supervisor chỉ route được nhưng không tạo ra evidence/policy/answer, và Sprint 4 không có dữ liệu trace để đánh giá.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi phụ thuộc vào Supervisor Owner để routing chọn đúng worker, và Trace & Docs Owner để dùng trace phản hồi lại các “điểm sai” trong contract/IO (nếu phát hiện mismatch).

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ cải tiến retrieval để **index hóa dữ liệu docs ngay trong repo nếu Chroma chưa có**, thay vì chỉ cảnh báo “chưa build index”. Bằng chứng là trace hiện cho thấy nhiều câu hỏi rơi vào nhánh retrieval nhưng `retrieved_chunks=[]` khi thiếu `chromadb`, dẫn tới confidence ~0.1. Tôi sẽ thêm một chế độ fallback “simple text search” trên `data/docs/*.txt` để pipeline vẫn có evidence tối thiểu khi demo.

