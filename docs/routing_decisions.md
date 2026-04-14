# Routing Decisions Log — Lab Day 09

**Nhóm:** Nhóm 36
**Ngày:** 2026-04-14

> **Hướng dẫn:** Ghi lại ít nhất **3 quyết định routing** thực tế từ trace của nhóm.
> Không ghi giả định — phải từ trace thật (`artifacts/traces/`).
> 
> Mỗi entry phải có: task đầu vào → worker được chọn → route_reason → kết quả thực tế.

---

## Routing Decision #1

**Task đầu vào:**
> "SLA xử lý ticket P1 là bao lâu?"

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `retrieval keyword matched: [p1, sla, ticket]`  
**MCP tools được gọi:** Không có  
**Workers called sequence:** `retrieval_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): "SLA xử lý ticket P1 là 4 giờ kể từ khi ticket được tạo. Phản hồi ban đầu là 15 phút. [1]"
- confidence: 0.59
- Correct routing? **Yes**

**Nhận xét:** _(Routing này đúng hay sai? Nếu sai, nguyên nhân là gì?)_

Định tuyến đúng. Câu hỏi truy vấn chi tiết về SLA, không liên quan đến kiểm tra quyền truy cập hay quy định hoàn tiền phức tạp nên được đưa thẳng tới `retrieval_worker`. Trả lời chính xác và đúng tài liệu `sla_p1_2026.txt`.

---

## Routing Decision #2

**Task đầu vào:**
> "Khách hàng có thể yêu cầu hoàn tiền trong bao nhiêu ngày?"

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `policy/access keyword matched: [hoàn tiền]`  
**MCP tools được gọi:** Không có (Tuy nhiên, flag `needs_tool: true` được bật sẵn)  
**Workers called sequence:** `retrieval_worker`, `policy_tool_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): "Khách hàng có thể yêu cầu hoàn tiền trong vòng 7 ngày làm việc... Ngoại lệ không được hoàn tiền bao gồm..."
- confidence: 0.48
- Correct routing? **Yes**

**Nhận xét:**

Định tuyến đúng. Supervisor phát hiện từ khoá "hoàn tiền" và nhận diện đây là một câu hỏi liên quan đến policy, yêu cầu `policy_tool_worker` xem xét kĩ các ngoại lệ (exception). Việc gọi mồi `retrieval_worker` trước khi vào `policy_tool` đảm bảo Agent có đủ fact để check rule.

---

## Routing Decision #3

**Task đầu vào:**
> "ERR-403-AUTH là lỗi gì và cách xử lý?"

**Worker được chọn:** `human_review` (HITL)  
**Route reason (từ trace):** `unknown error code + risk_high → human review | human approved → retrieval`  
**MCP tools được gọi:** Không có  
**Workers called sequence:** `human_review`, `retrieval_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): "Không đủ thông tin trong tài liệu nội bộ về lỗi ERR-403-AUTH và cách xử lý."
- confidence: 0.30
- Correct routing? **Yes**

**Nhận xét:**

Đây là định tuyến chính xác với kịch bản độ rủi ro cao. Phát hiện thấy format "ERR-xxx" lạ lẫm, Supervisor ngay lập tức flag `risk_high: true` và `hitl_triggered: true`. Flow bị tạm dừng cho `human_review`. Sau khi Approve, do tài liệu không có đề cập đến lỗi này, final answer đã abstain ("Không đủ thông tin") với mốc confidence thấp (0.3). Rất an toàn, tránh Hallucination.

---

## Tổng kết

### Routing Distribution

Từ kết quả chạy thực tế (traces run):

| Worker             | Số câu được route | % tổng                                           |
| ------------------ | ----------------- | ------------------------------------------------ |
| retrieval_worker   | 8 / 15            | 53%                                              |
| policy_tool_worker | 7 / 15            | 47%                                              |
| human_review       | 1 / 15            | 6% (Route ban đầu, sau đó chuyển sang retrieval) |

### Routing Accuracy

> Trong số X câu nhóm đã chạy, bao nhiêu câu supervisor route đúng?

- Câu route đúng: **15 / 15** (Dựa trên rule tĩnh set up sẵn).
- Câu route sai: 0 (Chưa phát hiện sai lệch rõ rệt so với bộ câu hỏi do đã bắt trúng keyword).
- Câu trigger HITL: **1** (Câu về lỗi lạ ERR-403-AUTH).

### Lesson Learned về Routing

> Quyết định kỹ thuật quan trọng nhất nhóm đưa ra về routing logic là gì?  

1. **Hierarchy của Routing (Rule priority):** Chúng tôi nhận thấy việc sắp xếp thứ tự ưu tiên của Keyword Matching cực kỳ quan trọng. Policy/Access/Risk phải được đè lên đầu, rồi mới đến Ticket, cuối cùng rơi vào fallback là Retrieval.
2. **Flagging (Cờ báo):** Không chỉ gán Route, Supervisor còn đánh các flag như `needs_tool` hoặc `risk_high`. Nhờ vậy, `policy_tool_worker` tuy là một nhưng hành vi linh hoạt: có câu thì gọi MCP, câu thì chỉ dò rule nội bộ mà thôi tuỳ biến theo Flag.

### Route Reason Quality

> Nhìn lại các `route_reason` trong trace — chúng có đủ thông tin để debug không?  
> Nếu chưa, nhóm sẽ cải tiến format route_reason thế nào?

Các log `route_reason` hiện tại (ví dụ: `route_reason: policy/access keyword matched: [hoàn tiền]`) tương đối đầy đủ và cực kỳ dễ hiểu đối với Human. Tuy nhiên, nó nên có thêm một field "Not Matched" liệt kê sơ những nhánh nó đã loại bỏ. Bằng cách đó, nếu route sai vào fallback, ta có thể biết nó đã trượt do miss cụm từ nào.
