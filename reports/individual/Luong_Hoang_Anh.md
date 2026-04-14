# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Lương Hoàng Anh  
**Vai trò trong nhóm:** Trace & Docs Owner  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `eval_trace.py`, các file tài liệu trong thư mục `docs/` (`system_architecture.md`, `routing_decisions.md`, `single_vs_multi_comparison.md`), và `reports/group_report.md`.
- Functions tôi implement: `analyze_traces()`, `compare_single_vs_multi()`, và luồng ghi file log `save_eval_report()`.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Vai trò của tôi diễn ra ở Sprint 4, tức chốt chặn cuối cùng. Tôi cần đợi các bạn làm API, Worker và Supervisor xong để có thể kích hoạt file chạy hàng loạt `eval_trace.py` trên tập `test_questions.json`. Kết quả chạy (traces) sinh ra trong `artifacts/traces/` chính là "bằng chứng thép" để tôi đóng gói thành Metric báo cáo và so sánh sự tối ưu của Multi-agent so với model cũ của Day 08.

**Bằng chứng:** 
Toàn bộ nội dung phân tích tự động trong file `artifacts/eval_report.json` và bảng so sánh Performance trong `single_vs_multi_comparison.md` do tôi trực tiếp viết.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Bổ sung thêm các trường đo lường `confidence` và `latency_ms` vào trong `AgentState` để Trace in ra chi tiết hơn, phục vụ cho việc Compare với Day 08.

**Lý do:**
Chỉ lưu lại output (câu trả lời) là không đủ để chứng minh Supervisor-Worker hữu ích. Việc đo thời gian chạy của cả cụm (latency) và yêu cầu `synthesis_worker` ghi nhận độ tự tin (confidence) sẽ giúp tạo thước đo đánh giá trực quan (ví dụ trung bình là ~3000ms tính ra Cost API cho System là cao). 

**Trade-off đã chấp nhận:**
Khối lượng dữ liệu lưu chuyển giữa các Worker (Shared State) lớn hơn một chút. Quá trình trace lưu file JSON to và dài hơn, nhưng bù lại mang đến cái nhìn Debug rành mạch.

**Bằng chứng từ trace/code:**
Trong trace log `run_20260414_161559.json` tôi export ra có rõ mốc đo lường:
```json
  "confidence": 0.59,
  "latency_ms": 3893,
  "timestamp": "2026-04-14T16:15:59.438395"
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Trace JSON bung rác kí tự Unicode (UnicodeEncodeError) làm Pipeline sập giữa chừng khi in ra terminal ở Windows. Lỗi đặc trưng khi thao tác xử lý file Encoding trên môi trường Windows.

**Symptom (pipeline làm gì sai?):**
Khi chạy vòng lặp đánh giá `python eval_trace.py` tới những câu thứ 2, thứ 3 có chứa tiếng Việt như lỗi "Khách hàng mua Flash Sale...". Terminal lập tức in Exception và văng khỏi Pipeline, Trace JSON sinh ra bị cụt hoặc ghi dạng kí tự mã hoá \u00xx.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**
Lỗi ở tầng Python Standard Output không tự động cast về `UTF-8` khi tương tác console trên Windows và `json.dump` thiếu param báo hiệu định dạng tiếng Việt.

**Cách sửa:**
Inject config reconfigure `sys.stdout` ngay từ đầu chương trình `eval_trace.py`. Xử lý lưu `json.dump` với cờ `ensure_ascii=False`.

**Bằng chứng trước/sau:**
```python
# Đoạn code trong eval_trace.py tôi sử dụng để Fix:
if hasattr(sys.stdout, "reconfigure"):
    sys.stdout.reconfigure(encoding="utf-8", errors="replace")
    
# Và khi Write file Report:
with open(output_file, "w", encoding="utf-8") as f:
    json.dump(comparison, f, ensure_ascii=False, indent=2)
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tổng hợp Data. Khả năng nhìn nhận Trace Log khô khan biến thành những bảng metric so sánh rõ rệt giữa Day 08 và Day 09, từ đó nêu bật được điểm mạnh của cấu trúc Agent độc lập qua từng node (Tính Modular) và sự lợi hại của MCP Server khi cắm Tool. Đảm bảo toàn bộ nhóm nộp đủ Docs và Group Report trơn tru.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Phần Code logic đan kết các Core AI như (LLM Inference hay Vector Embedding Search) thì tôi chưa nắm quá rành, hoàn toàn phụ thuộc vào file chức năng các bạn kia đưa nên bản thân ko tự debug được nếu module Worker tịt ngòi báo lỗi API.

**Nhóm phụ thuộc vào tôi ở đâu?** 
Quá trình Eval Report và Document so sánh. Nếu tôi không chạy xong thì không có Metric cho các bạn điền vào Báo cáo cá nhân hay nộp lấy điểm trọn vẹn Lab.

**Phần tôi phụ thuộc vào thành viên khác:** 
Phụ thuộc hoàn toàn 3 Sprint đầu. Team A, B, C phải nối thành công Pipeline thì tôi mới có Engine để thả tập `test_questions.json` vào càn quét lấy Trace Log.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ tích hợp Evaluator-as-a-judge (Sử dụng 1 model LLM nhỏ) thay vì Compare thủ công bằng Text. Lúc đó LLM sẽ tự động móc nối file `test_questions.json` để kiểm tra Final Answer coi có bám sát context kì vọng không và tự cấp Score cho từng Trace. Trace của câu `q09` cho thấy tự kiểm tra Answer Text bằng code tay trên JSON đang rất khó vì model trả về cách viết câu từ đa dạng (Không cố định "có" hoặc "không").
