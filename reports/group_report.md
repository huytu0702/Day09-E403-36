# Báo Cáo Nhóm — Lab Day 09: Multi-Agent Orchestration

**Tên nhóm:** Nhóm 36  
**Thành viên:**
| Tên                | Vai trò          | Email           |
| ------------------ | ---------------- | --------------- |
| Nguyễn Huy Tú      | Supervisor Owner | nguyenhuytu724@gmail.com |
| Phạm Quốc Vương    | Worker Owner     | phamvuong2622004@gmail.com |
| Trương Minh Phuớc  | MCP Owner        | tv_c@domain.com |
| Nguyễn Thành Trung | Trace Owner      | tv_d@domain.com |
| Lương Hoàng Anh    | Docs Owner       | hitomisaichi@gmail.com |

**Ngày nộp:** 2026-04-14  
**Repo:** https://github.com/huytu0702/Day09-E403-36


---

## 1. Kiến trúc nhóm đã xây dựng (150–200 từ)

**Hệ thống tổng quan:**

Nhóm chúng tôi triển khai mẫu Supervisor-Worker chia tác vụ RAG thành 3 Worker nòng cốt để chịu trách nhiệm riêng biệt:
- **Retrieval Worker**: Chuyên trách nhúng câu hỏi người dùng (Embed query) và tìm kiếm Vector DB để lấy các Chunk tài liệu gần nhất.
- **Policy Tool Worker**: Xử lý logic cứng như policy ngoại lệ mà LLM khó tự suy ra. Worker này có kết nối với kho Tool bên ngoài qua MCP protocol.
- **Synthesis Worker**: Chặn cuối, tóm gọn logic thu nhặt từ cả Retrieval và Policy thành câu trả lời hoàn chỉnh dựa trên ngôn ngữ tự nhiên. 
Tất cả công đoạn này được điều phối hướng đi qua một Node chính giữa là **Supervisor Node**. Node này đóng vai trò Router để đưa câu hỏi vào luồng chạy phù hợp.

**Routing logic cốt lõi:**
Logic mà Supervisor đang dùng là Keyword Matching kết hợp Regex cho mã lỗi lạ.
Nó kiểm duyệt mảng keywords như "policy, hoàn tiền, cấp quyền, flash sale..." để route cho `policy_tool_worker`. Hoặc nếu là "p1, SLA, incident" thì trả cho `retrieval_worker`. Đối với từ khoá chứa Risk cao như "urgently" hoặc Mã code không rõ (ví dụ: `[ERR_xxx]`), nó kích hoạt Cờ Hitl để Human_Review. Cuối cùng, những câu lạc đề được đẩy lại Retrieval default.

**MCP tools đã tích hợp:**
- `search_kb`: Công cụ thay thế Retrieval mặc định để Policy Agent tự bươi móc tài liệu nếu Supervisor "quên" Route cho Retrieval Flow đầu tiên.
- `get_ticket_info`: Lấy data thực (Mock json) cho các truy vấn về ticket như `P1-LATEST`.
- `check_access_permission`: Cấp quyền truy xuất Level 1 2 3 tích hợp các quy trình phê chuẩn. Nhờ MCP, Agent không cần phải "Nhớ" logic cấp quyền mà chỉ việc ném Parameter qua API và lấy về kết quả Trạng Thái.

---

## 2. Quyết định kỹ thuật quan trọng nhất (200–250 từ)

**Quyết định:** Chế độ rẽ nhánh bắt buộc Route qua Retrieval trước khi Policy_Tool_Worker Check.

**Bối cảnh vấn đề:**
Ở Sprint 1, nếu Supervisor thấy từ khoá liên quan tới Policy, nó nhảy luôn vào `policy_tool_worker`. Tuy nhiên, vì Supervisor không cung cấp document context, Policy Tool không biết policy hoàn tiền là gì để check, nên kết quả Policy Check bị sai. Nhóm phải quyết có nên cấy MCP DB search thẳng vào Policy luôn hay bắt Supervisor định tuyến luồng tuần tự.

**Các phương án đã cân nhắc:**

| Phương án                                                      | Ưu điểm                                                                                                                                | Nhược điểm                                                                                                                                           |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Gắn ChromaDB vào thẳng Policy Worker                           | Có thể lấy context tại chỗ rất nhanh (đỡ phải route thêm Node)                                                                         | Code bị Monolithic dần, vi phạm nguyên tắc SRP (Single Responsibility Principle) của các Agent. Policy Tool lại đi kiêm luôn nhiệm vụ của Retrieval. |
| Yêu cầu Supervisor xếp vòng `Retrieval -> Policy -> Synthesis` | Module hoá hoàn toàn. Mọi người code trong file riêng không dẫm chân lên nhau. Khúc sau nếu thay thư viện Vector Database sẽ nhàn rỗi. | Mất chút thời gian đi qua vòng lặp check. Pipeline xử lý luồng có vẻ cồng kềnh hơn khoảng 30%.                                                       |

**Phương án đã chọn và lý do:**
Bắt Supervisor route tuần tự: Ưu tiên có evidence trước, rồi mới check exceptions. Việc này giữ tính Module hóa cực cao. Thể hiện qua `graph.py` của Supervisor. Nếu route là policy_tool_worker, nó sẽ force state qua `retrieval_worker` nếu state chunk đang trống rỗng.

**Bằng chứng từ trace/code:**
```python
        elif route == "policy_tool_worker":
            # Policy questions: ưu tiên có evidence trước, rồi mới policy check
            if not state.get("retrieved_chunks"):
                state = retrieval_worker_node(state)
            state = policy_tool_worker_node(state)
```
 Trace Log (File run_20260414_161603):
```json
  "supervisor_route": "policy_tool_worker",
  "workers_called": [
    "retrieval_worker",
    "policy_tool_worker",
    "synthesis_worker"
  ]
```

---

## 3. Kết quả grading questions (150–200 từ)

**Tổng điểm raw ước tính:** 96 / 96

**Câu pipeline xử lý tốt nhất:**
- ID: `q13` (Test câu hỏi: Yêu cầu cấp quyền Level 3 khẩn cấp 2am) — Lý do tốt: Node Supervisor đã thực thi rất đầy đủ (Phát hiện Risk -> Mồi Retrieval lấy SLA rule -> Gọi MCP `check_access_permission` -> Human Review Hitl). Agent Synthesis xuất sắc tổng hợp rành mạch các Approval.

**Câu pipeline fail hoặc partial:**
- ID: N/A — Cấu trúc hiện tại đáp ứng chuẩn 15 test câu hỏi public đề ra.  

**Câu gq07 (abstain):** Nhóm xử lý thế nào?
Để Synthesis hoạt động an toàn, trong System prompt bắt gọt câu trả lời phải nằm trong Evidence. Nếu Evidence trả rỗng thì nó báo lỗi. Nhưng để an toàn hơn, Worker Rule của Supervisor khi gặp dạng câu hỏi lệch đã bắn Cờ `risk_high: true` -> Human Review. Sau đó Abstain.

**Câu gq09 (multi-hop khó nhất):** Trace ghi được 2 workers không? Kết quả thế nào?
Trace của một câu khó (Cần Retrieval lấy rule trước, xong Policy bóc rule ra, gửi cái mớ đó cho Synthesis gộp chung). Trace Log in ra tới 3 Node rành mạch. Confidence ~ 0.5 - 0.7 nhưng độ logic câu chữ khá cao, không bị ảo giác.

---

## 4. So sánh Day 08 vs Day 09 — Điều nhóm quan sát được (150–200 từ)

**Metric thay đổi rõ nhất (có số liệu):**
Thời gian chạy đội lên đáng khế. `latency` tăng từ khoảng 1,5s lên đến ngưỡng `~3800ms` với một câu khá đơn giản (Ví dụ như hỏi SLA P1). LLM Call tăng giá API.

**Điều nhóm bất ngờ nhất khi chuyển từ single sang multi-agent:**
Việc "Lỗi xuất hiện ở đâu" trở nên dễ kiếm cực kì. Khi Output trả về câu trả lời lẩm cẩm, chỉ cần bật File Json xem. Nếu Exception Count là 0 tức là Policy Code bị ngu, nếu Chunk Count là 0 tức là Database tìm kiếm gà.

**Trường hợp multi-agent KHÔNG giúp ích hoặc làm chậm hệ thống:**
Khi User gửi câu rắp tâm hỏi nhảm nhí, chitchat. Model Single Agent có thể đáp lẹ trong 0,5 giây. Với Multi Agent Supervisor bắt buộc phải càn qua một cụm Keyword Regex khổng lồ -> đẩy nó sang vòng vòng 1 lũ Agent rồi trả về câu trả lời rỗng với độ khó chịu vô cùng (Latency cao chỉ để nhận Abstain).

---

## 5. Phân công và đánh giá nhóm (100–150 từ)

**Phân công thực tế:**

| Thành viên         | Phần đã làm                                        | Sprint   |
| ------------------ | -------------------------------------------------- | -------- |
| Nguyễn Huy Tú      | `graph.py` Xây dựng cấu hình Agent Router          | Sprint 1 |
| Phạm Quốc Vương    | `workers/*` Viết Logic xử lý cho từng Agent đơn lẻ | Sprint 2 |
| Truưng Minh Phuớc  | `mcp_server.py` Viết API giả lập                   | Sprint 3 |
| Nguyễn Thành Trung | `eval_trace` Kiểm thử và kết xuất Trace            | Sprint 4 |
| Lương Hoàng Anh    | Đóng gói Review & Điền Báo Cáo                     | Sprint 4 |

**Điều nhóm làm tốt:**
Chia Sprint cực kỳ gọn và tôn trọng Abstract Class Contract. Team B code Worker chỉ cần định nghĩa Interface. Team A nối file tự động trơn tru.

**Điều nhóm làm chưa tốt hoặc gặp vấn đề về phối hợp:**
MCP Server (FastApi) chưa thể test End-to-end tốt vì phải liên tục bật 2 Terminal làm mất thời gian đồng bộ môi trường.

**Nếu làm lại, nhóm sẽ thay đổi gì trong cách tổ chức?**
Sử dụng một Framework điều phối Node xịn xò với UI trực tiếp như LangGraph từ đầu thay vì code luồng If-Else truyền thống mệt mỏi này để hiển thị UI trực quan rẽ nhánh Graph.

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì? (50–100 từ)

Chúng tôi sẽ sử dụng LLM để thay thế Logic Router Keyword. Nếu dùng LLM làm Classifier Supervisor thì Model sẽ có tính ngữ nghĩa Semantic hơn, route chính xác 99% đối với những trường hợp Keyword bị đồng nghĩa thay vì rơi hết vào Default Fallback (Ví dụ người ta ghi "trả tiền lại" thay vì "hoàn tiền"). Kèm theo LangGraph State phục hồi Re-Routing (Self Reflection).

---

*File này lưu tại: `reports/group_report.md`*  
*Commit sau 18:00 được phép theo SCORING.md*
