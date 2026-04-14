# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Trương Minh Phước  
**Vai trò trong nhóm:** MCP Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

> **Lưu ý quan trọng:**
> - Viết ở ngôi **"tôi"**, gắn với chi tiết thật của phần bạn làm
> - Phải có **bằng chứng cụ thể**: tên file, đoạn code, kết quả trace, hoặc commit
> - Nội dung phân tích phải khác hoàn toàn với các thành viên trong nhóm
> - Deadline: Được commit **sau 18:00** (xem SCORING.md)
> - Lưu file với tên: `reports/individual/[ten_ban].md` (VD: `nguyen_van_a.md`)

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `mcp_server.py` và `workers/policy_tool.py`.
- Functions tôi implement: Xây dựng server bằng FastAPI (`api_list_tools`, `api_call_tool`), cấu hình `urllib.request` để client gọi API `_call_mcp_tool`.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Phần việc của tôi đóng vai trò cầu nối dữ liệu bên ngoài hệ thống cho Worker. `Policy Tool Worker` do Worker Owner quản lý cần các khả năng tra cứu như `search_kb` hay `get_ticket_info` để có bối cảnh kiểm tra rules. Tôi cung cấp interface HTTP độc lập để Policy Tool có thể gọi qua REST thay vì hardcode. Việc này phục vụ trực tiếp cho quá trình tổng hợp (synthesis) cuối cùng của hệ thống.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**
Sửa đổi chính được áp dụng lên nhánh implement MCP:
- Bổ sung `FastAPI` instance tại `mcp_server.py`.
- Cập nhật hàm `def _call_mcp_tool(tool_name: str, tool_input: dict) -> dict:` trong `policy_tool.py`.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Nâng cấp Mock MCP Server thành chuẩn **HTTP Server chạy qua FastAPI** kết hợp cơ chế **Graceful Fallback** để giao tiếp (Advanced Option).

**Lý do:**
Yêu cầu ban đầu ở mức Standard cho phép tôi chỉ cần gọi tool bằng cách `import` class nội bộ từ file. Nhưng tôi nhận thấy thực tế mô hình MCP (Model Context Protocol) yêu cầu server và agent decoupled hoàn toàn để có thể dễ dàng scale và uỷ quyền truy cập ở các môi trường phân tán. Do đó, tôi đã lập trình một API server bằng FastAPI ở cổng `:8000`.

**Trade-off đã chấp nhận:**
Điểm đánh đổi là nếu service bị crash hoặc chưa start port 8000, pipeline có thể bị gián đoạn.

**Bằng chứng từ trace/code:**
Để xử lý điểm đánh đổi trên, tôi đã thêm Graceful Fallback vào logic `_call_mcp_tool`. Nếu không thể POST được JSON tới HTTP Server thì sẽ back lại import trực tiếp:
```python
    try:
        # Thử gọi qua HTTP (Advanced)
        url = "http://localhost:8000/tools/call"
        req = urllib.request.Request(url, method="POST")
        req.add_header('Content-Type', 'application/json')
        # ... logic parse
    except Exception as http_e:
        # Graceful Fallback (Standard) nếu Uvicorn không start
        from mcp_server import dispatch_tool
        result = dispatch_tool(tool_name, tool_input)
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Dịch vụ API MCP không thể gửi/nhận dữ liệu linh hoạt và worker không bắt được lỗi nếu tool không tồn tại.

**Symptom (pipeline làm gì sai?):**
Nếu Supervisor gọi một Tool (như `create_ticket`) nhưng parameter truyền vào sai chuẩn schema hoặc không match, in-memory function call báo lỗi Type Error hoặc stack trace python, khiến toàn bộ pipeline vỡ và trả về lỗi 500 cho ứng dụng người dùng cuối. 

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**
Root cause nằm ở Worker logic khi không bắt giữ format json và schema của input dict khi client gọi, đồng thời gọi trực tiếp thiếu wrapper HTTP dẫn đến không quản trị được Exception.

**Cách sửa:**
Trong hàm `dispatch_tool`, tôi đã bọc ngoại lệ `TypeError` làm Error Code Response đồng thời trả về kèm theo Input Schema bắt buộc:
```python
    except TypeError as e:
        return {
            "error": f"Invalid input for tool '{tool_name}': {e}",
            "schema": TOOL_SCHEMAS[tool_name]["inputSchema"],
        }
```
Tại FastAPI, tôi chủ động Raise ra `HTTPException(status_code=400, detail=result["error"])`.

**Bằng chứng trước/sau:**
Kết quả trả về trace đã thể hiện: `{"error": "Invalid input for tool 'create_ticket': ...", "schema": ...}` thay vì crash văng ra khỏi luồng pipeline.

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi đã bao quát tốt kiến trúc hệ thống và mang lại chuẩn MCP protocol qua Fast API, bảo đảm dự án đạt được điểm "Advanced Bonus" như cấu trúc Lab Day 09 đề ra.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Thiết kế Schema cho các function/tool bằng Type Hint và Pydantic để model tự sinh Schema vẫn chưa được tối ưu, hiện vẫn đang phải hardcode biến `TOOL_SCHEMAS`.

**Nhóm phụ thuộc vào tôi ở đâu?**
`policy_tool` worker và `synthesis_tool` hoàn toàn bị crash hoặc không lấy được context bổ sung nếu MCP server của tôi chưa thể đáp ứng API request.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi cần "Worker Owner" hoàn thiện prompt và LLM chain trong worker để agent tự động điền các thông tin (ví dụ `ticket_id`) đúng định dạng trước khi POST xuống HTTP server MCP.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ xoá lớp Wrapper Mock HTTP API tự viết bằng FastAPI và thay bằng SDK `fastmcp` (thư viện chính thức của Anthropic dành cho MCP Server). Bằng chứng ở tài liệu kiến trúc cho thấy việc tự map dict `TOOL_SCHEMAS` rất rủi ro. `fastmcp` sử dụng hàm trang trí `@mcp.tool()` giúp mô tả input param chuẩn hoá cho LLM context mà không cần viết JSON raw.
