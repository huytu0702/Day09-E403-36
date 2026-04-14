# Single Agent vs Multi-Agent Comparison — Lab Day 09

**Nhóm:** Nhóm 36  
**Ngày:** 2026-04-14

> **Hướng dẫn:** So sánh Day 08 (single-agent RAG) với Day 09 (supervisor-worker).
> Phải có **số liệu thực tế** từ trace — không ghi ước đoán.
> Chạy cùng test questions cho cả hai nếu có thể.

---

## 1. Metrics Comparison

> Điền vào bảng sau. Lấy số liệu từ:
> - Day 08: chạy `python eval.py` từ Day 08 lab
> - Day 09: chạy `python eval_trace.py` từ lab này

| Metric                | Day 08 (Single Agent)    | Day 09 (Multi-Agent) | Delta       | Ghi chú                                           |
| --------------------- | ------------------------ | -------------------- | ----------- | ------------------------------------------------- |
| Avg confidence        | N/A                      | 0.517                | N/A         | Ngày 8 đo nguyên khối, không tính điểm Confidence |
| Avg latency (ms)      | ~1000 - 2000 (Ước lượng) | 3089                 | + ~1500ms   | Day 09 tốn chừng 3-4s do nối đuôi Workers         |
| Abstain rate (%)      | N/A                      | 6% (1/15)            | N/A         | % câu trả về "không đủ info" (Abstain)            |
| Multi-hop accuracy    | N/A                      | Xử lý tốt (q13, q15) | N/A         | Multi-agent giúp đối chiếu policy và fact tốt hơn |
| Routing visibility    | ✗ Không có               | ✓ Có route_reason    | N/A         | Giúp Debug nhanh                                  |
| Debug time (estimate) | 15 - 20 phút             | ~ 5 phút             | -10-15 phút | Thời gian tìm ra 1 bug (Theo Json Analysis)       |

---

## 2. Phân tích theo loại câu hỏi

### 2.1 Câu hỏi đơn giản (single-document)

| Nhận xét    | Day 08                                     | Day 09                                                                                    |
| ----------- | ------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Accuracy    | Tốt                                        | Tốt (Hoàn toàn chính xác)                                                                 |
| Latency     | ~1.5s                                      | 3.0s - 3.8s                                                                               |
| Observation | Truy xuất nhanh vì chỉ dùng prompt một lần | Route qua retrieval_worker mất chút thời gian nhưng phân phối chuyên biệt hoá cho context |

**Kết luận:** Multi-agent có cải thiện không? Tại sao có/không?

Ở các câu hỏi quá đơn giản, hệ AI 2 lớp (Supervisor - Worker) khiến Pipeline bị chậm hơn mà Accuracy không chênh lệch (Cả 2 đều đúng). Sự trả giá nằm ở Latency.

### 2.2 Câu hỏi multi-hop (cross-document)

| Nhận xét         | Day 08                                                              | Day 09                                                             |
| ---------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Accuracy         | Không ổn định                                                       | Rất tốt (Ví dụ q13, q15)                                           |
| Routing visible? | ✗                                                                   | ✓                                                                  |
| Observation      | LLM bị nhồi một đống chunk từ nhiều rule vả đá nhau dễ sinh ảo giác | Worker được call cụ thể phối hợp Retrieval và Policy Tool triệt để |

**Kết luận:**

Cải thiện rất mạnh. Vì `policy_tool_worker` tách riêng quá trình trích xuất Exception (được Code sẵn/LLM trích xuất rõ), nhờ vậy `synthesis_worker` (nhận câu trả lời cuối) có cái nhìn rành rọt nhất trước khi generate cuối.

### 2.3 Câu hỏi cần abstain

| Nhận xét            | Day 08                                         | Day 09                                                               |
| ------------------- | ---------------------------------------------- | -------------------------------------------------------------------- |
| Abstain rate        | N/A (Có khả năng ảo giác)                      | 6%                                                                   |
| Hallucination cases | Có rủi ro                                      | Không có                                                             |
| Observation         | Các mã lỗi "lạ" (ERR-403) có thể khiến LLM bịa | Supervisor có cơ chế check Unknown error đặng gọi node HITL giải cứu |

**Kết luận:**

Thiết kế Multi-Agent cho phép supervisor nhấc cờ "Báo Động" khi thấy nguy hiểm, kích hoạt Human-in-the-loop (HITL) chặn ảo giác lại tức thì tại đầu nguồn trước khi Generate.

---

## 3. Debuggability Analysis

> Khi pipeline trả lời sai, mất bao lâu để tìm ra nguyên nhân?

### Day 08 — Debug workflow
```
Khi answer sai → phải đọc toàn bộ RAG pipeline code → tìm lỗi ở indexing/retrieval/generation
Không có trace → không biết bắt đầu từ đâu
Thời gian ước tính: 15-20 phút
```

### Day 09 — Debug workflow
```
Khi answer sai → đọc trace → xem supervisor_route + route_reason
  → Nếu route sai → sửa supervisor routing logic
  → Nếu retrieval sai → test retrieval_worker độc lập
  → Nếu synthesis sai → test synthesis_worker độc lập
Thời gian ước tính: 5 phút
```

**Câu cụ thể nhóm đã debug:** _(Mô tả 1 lần debug thực tế trong lab)_

Để kiểm tra xem vì sao câu hỏi về Flash Sale lại không được hoàn tiền, thay vì chạy nguyên cục Pipeline, chúng tôi mở file trace `run_20260414_161603.json`. Phát hiện Worker gọi là `policy_tool_worker`. Chỉ cần chạy độc lập file Policy Tool với Context là có thể thấy ngay Flag Exceptions được Raise. Thời gian kiểm lỗi trong vòng vài chục giây.

---

## 4. Extensibility Analysis

> Dễ extend thêm capability không?

| Scenario                    | Day 08                         | Day 09                                            |
| --------------------------- | ------------------------------ | ------------------------------------------------- |
| Thêm 1 tool/API mới         | Phải sửa toàn prompt           | Thêm MCP tool + route rule                        |
| Thêm 1 domain mới           | Phải retrain/re-prompt         | Thêm 1 worker mới                                 |
| Thay đổi retrieval strategy | Sửa trực tiếp trong pipeline   | Sửa retrieval_worker độc lập                      |
| A/B test một phần           | Khó — phải clone toàn pipeline | Dễ — swap worker (Ví dụ test 2 cái policy worker) |

**Nhận xét:**

Kiến trúc Agent chia nhỏ giải quyết bài toán trừu tượng cục bộ hoàn toàn – mọi module đều là Black box với Component khác qua IO Contract. Khả năng mở rộng gần như là Vô Hạn nếu máy chủ đáp ứng được.

---

## 5. Cost & Latency Trade-off

> Multi-agent thường tốn nhiều LLM calls hơn. Nhóm đo được gì?

| Scenario      | Day 08 calls | Day 09 calls                  |
| ------------- | ------------ | ----------------------------- |
| Simple query  | 1 LLM call   | 1-2 LLM calls (Kèm Embedding) |
| Complex query | 1 LLM call   | 2-3 LLM calls                 |
| MCP tool call | N/A          | +1 cho mỗi MCP cần Tool Check |

**Nhận xét về cost-benefit:**

Tăng gấp rưỡi tới gấp đôi hệ số LLM API. Tuy vậy, trong Enterprise Grade App cho Helpdesk, độ chính xác và Zero-Hallucination là thước đo tiên quyết thay vì chi phí quá nhỏ cho tiền API hiện tại. Sự đánh đổi là chấp nhận được.

---

## 6. Kết luận

> **Multi-agent tốt hơn single agent ở điểm nào?**

1. Khả năng gỡ lỗi hệ thống (Debuggability) tăng vọt. Rành mạch luồng Routing.
2. An toàn cao. Hạn chế đứt đoạn Pipeline và có thể đưa Cảnh báo, tích hợp Tool cho các câu phức tạp ngoại lệ.

> **Multi-agent kém hơn hoặc không khác biệt ở điểm nào?**

1. Thời gian gọi hàm dông dài và tốn LLM API Token lớn hơn trên mặt bằng chung.
2. Về các câu hỏi Simple Retrieval thì Single Agent hay Multi đều hoạt động như nhau (Vì Multi Agent cuối cũng Route về Retrieval Worker chứ chả làm bài test gì quá phức tạp). 

> **Khi nào KHÔNG nên dùng multi-agent?**

Hệ thống cần Fast-respond real-time (dưới 1s), hoặc Data Context có dạng cực kì nông không mang tính chất phân nhánh rẽ hướng, ví dụ "Trợ lý tra cứu nội quy cơ bản" chả hạn.

> **Nếu tiếp tục phát triển hệ thống này, nhóm sẽ làm thêm gì?**

Chuyển đổi hoàn toàn sang LangGraph StateGraph kết nối Tool Nodes để Flow này chạy Re-Routing tự sửa chữa (Khắc phục lỗi Retrieval tạch khiến Synthesis không có info để Answer).
