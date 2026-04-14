# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Huy Tú  
**Vai trò trong nhóm:** Supervisor Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `graph.py`
- Functions tôi implement: `supervisor_node()` và phần cập nhật state phục vụ routing trong `AgentState`, `make_initial_state()`

Tôi phụ trách phần Supervisor của Sprint 1, cụ thể là logic route trong `graph.py`. Tôi bổ sung các nhóm từ khóa cho hai loại câu hỏi chính là policy/access và retrieval/SLA, thêm bước đánh dấu `risk_high`, nhận diện mã lỗi lạ bằng regex, rồi ghi kết quả vào state qua các field `supervisor_route`, `route_reason`, `needs_tool`, `risk_high`. Tôi cũng chỉnh lại format log trong `history` để quyết định route đọc lại rõ hơn. Phần việc này nối trực tiếp với các worker: nếu supervisor route sai thì retrieval, policy và synthesis đều đi sai hướng ngay từ đầu.

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Tôi tạo đầu vào điều phối cho các worker. Nếu `supervisor_node()` route sai hoặc không ghi đủ lý do vào state, các bạn phụ trách worker và trace sẽ rất khó debug toàn pipeline.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi chọn routing theo keyword có thứ tự ưu tiên rõ ràng, kèm một lớp override sang `human_review` nếu task có mã lỗi lạ và rủi ro cao.

**Lý do:**

Ở Sprint 1, mục tiêu chính là supervisor phải route được và giải thích được, nên tôi không dùng LLM để classify. Tôi chia từ khóa thành ba nhóm: `policy_keywords`, `retrieval_keywords` và `risk_keywords`. Sau đó tôi áp dụng thứ tự ưu tiên: câu hỏi về hoàn tiền, cấp quyền, access level sẽ vào `policy_tool_worker`; câu hỏi về `p1`, `sla`, `ticket`, `incident` sẽ vào `retrieval_worker`; còn lại đi retrieval mặc định. Với mã lỗi kiểu `ERR-XXX` hoặc `ERR_XXX`, tôi dùng regex để nhận diện vì đây không phải một keyword cố định. Nếu task vừa có mã lỗi lạ vừa bị đánh dấu risk thì tôi override sang `human_review` để tránh worker tự trả lời khi chưa đủ context.

**Trade-off đã chấp nhận:**

Trade-off của cách này là supervisor phụ thuộc vào danh sách keyword tôi khai báo. Nếu user diễn đạt quá khác bộ từ khóa này thì route có thể chưa tối ưu. Bù lại, mọi quyết định đều giải thích được ngay bằng `route_reason`, phù hợp với mục tiêu trace của lab.

**Bằng chứng từ trace/code:**

```python
policy_keywords = [
    "hoàn tiền", "refund", "flash sale", "license",
    "cấp quyền", "access", "level 3", "policy", "quyền",
    "emergency",
]
retrieval_keywords = [
    "p1", "escalation", "sla", "ticket", "incident",
]
unknown_error_re = re.compile(r'\berr[-_]\w+', re.IGNORECASE)

if matched_policy:
    route = "policy_tool_worker"
elif matched_retrieval:
    route = "retrieval_worker"

if risk_high and has_unknown_error:
    route = "human_review"
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** `supervisor_node()` trước commit của tôi vẫn là placeholder nên graph chưa có routing logic thật cho Sprint 1.

**Symptom (pipeline làm gì sai?):**

Trước khi tôi sửa, phần `graph.py` mới chỉ có comment “TODO Sprint 1” cho routing. State đã có `route_reason`, `needs_tool`, `risk_high`, nhưng route thực tế vẫn gần như mặc định. Supervisor chưa phân biệt được câu hỏi hoàn tiền, câu hỏi SLA hay câu hỏi có mã lỗi bất thường. Điều đó làm supervisor chưa thực hiện đúng vai trò điều phối của Sprint 1.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**

Root cause nằm ở tầng routing của supervisor, không phải ở indexing hay worker. Hệ thống đã có khung `AgentState` và `history`, nhưng chưa có logic thật để điền các field quyết định.

**Cách sửa:**

Tôi thay phần placeholder bằng ba bước rõ ràng trong `supervisor_node()`: đánh dấu risk từ `risk_keywords` và regex mã lỗi, match keyword theo thứ tự ưu tiên policy trước retrieval, rồi override sang `human_review` nếu có `risk_high` và `has_unknown_error`. Tôi cũng sửa format log trong `history` thành `route=... | reason=...` để trace đọc dễ hơn.

**Bằng chứng trước/sau:**

```python
state["supervisor_route"] = route
state["route_reason"] = route_reason
state["needs_tool"] = needs_tool
state["risk_high"] = risk_high
state["history"].append(f"[supervisor] route={route} | reason={route_reason}")
```

Sau commit này, `graph.py` không còn route mặc định kiểu placeholder nữa mà đã có lý do route cụ thể theo từng task.

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Tôi làm tốt nhất ở việc biến phần supervisor từ khung sườn có comment TODO thành logic route thật, có thể đọc, kiểm tra và mở rộng tiếp ở các sprint sau. Tôi cũng giữ `route_reason` đủ cụ thể để trace không chỉ cho biết đi nhánh nào mà còn cho biết vì sao đi nhánh đó.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Điểm tôi còn yếu là routing hiện vẫn hoàn toàn rule-based. Nó phù hợp với lab và dễ debug, nhưng chưa linh hoạt nếu câu hỏi diễn đạt quá xa bộ từ khóa.

**Nhóm phụ thuộc vào tôi ở đâu?** _(Phần nào của hệ thống bị block nếu tôi chưa xong?)_

Nếu tôi chưa hoàn tất `supervisor_node()`, nhóm sẽ chưa thể thực hiện tiếp các sprint sau.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_

Tôi phụ thuộc vào Worker Owner để sau khi supervisor route xong, từng worker thực sự xử lý đúng contract; và phụ thuộc vào Trace & Docs Owner để dùng các field tôi ghi vào state khi viết phân tích routing.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ nâng cấp `supervisor_node()` từ rule-based thuần sang hybrid routing: vẫn giữ keyword priority hiện tại, nhưng bổ sung một lớp chấm điểm tín hiệu để xử lý tốt hơn các câu hỏi có nhiều ý cùng lúc như vừa có `P1` vừa có `access` hoặc `refund`. Cách này phù hợp với agent hiện tại vì không cần thêm LLM ở bước route, vẫn nhanh và trace được, nhưng quyết định sẽ mềm hơn so với `if/elif` cứng. Nếu làm được, supervisor sẽ route ổn định hơn khi câu hỏi dài và nhiều ngữ cảnh.
