# Báo cáo Kỹ thuật: Thiết lập Khế ước Chuyển giao Dữ liệu (Hand-off Contract Schema)

**Dự án:** Luồng Sales Intelligence tự động - Viettel Solutions
**Mã bài tập:** AI301_S08_BTVN_M2_BT1
**Tác giả:** [Tên của bạn]

---

## 1. Bối cảnh và Mục tiêu
Trong hệ thống Multi-Agent phục vụ Sales Intelligence, dữ liệu được luân chuyển qua 3 trạm: **Research Agent** (Nghiên cứu) -> **Scoring Agent** (Chấm điểm) -> **Proposal Agent** (Đề xuất). Để ngăn chặn hiện tượng "ảo giác" (hallucination), mất mát thông tin và lỗi định dạng khi chuyển giao từ Research Agent sang Scoring Agent, việc thiết lập một Khế ước bàn giao dữ liệu (Hand-off Contract) bằng JSON Schema là bắt buộc.

---

## 2. Nhiệm vụ 1: Định nghĩa cấu trúc JSON Schema (Hand-off Contract)

Khế ước này định nghĩa khắt khe kiểu dữ liệu, các trường bắt buộc nhằm đảm bảo Scoring Agent luôn nhận được một "gói tin" hoàn hảo để tính điểm tiềm năng khách hàng.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Research_To_Scoring_Handoff",
  "description": "Khế ước dữ liệu chuẩn từ Research Agent sang Scoring Agent",
  "type": "object",
  "properties": {
    "customer_id": {
      "type": "string",
      "pattern": "^VTS-[A-Z0-9]{6}$",
      "description": "Mã khách hàng định dạng chuẩn VTS-XXXXXX"
    },
    "company_name": {
      "type": "string",
      "minLength": 3,
      "description": "Tên pháp lý đầy đủ của doanh nghiệp"
    },
    "estimated_revenue_usd": {
      "type": "integer",
      "minimum": 0,
      "description": "Doanh thu ước tính quy đổi ra USD"
    },
    "company_size": {
      "type": "integer",
      "minimum": 1,
      "description": "Tổng số lượng nhân sự"
    },
    "tech_needs": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["Cloud", "AI", "CyberSecurity", "ERP", "IoT"]
      },
      "minItems": 1,
      "description": "Danh sách nhu cầu công nghệ (được chuẩn hóa)"
    },
    "decision_makers": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "role": { 
            "type": "string", 
            "enum": ["CIO", "CTO", "CEO", "IT Director"] 
          }
        },
        "required": ["name", "role"]
      },
      "minItems": 1,
      "description": "Danh sách nhân sự cấp cao ra quyết định"
    }
  },
  "required": [
    "customer_id", 
    "company_name", 
    "estimated_revenue_usd", 
    "company_size", 
    "tech_needs", 
    "decision_makers"
  ],
  "additionalProperties": false
}
```

---

## 3. Nhiệm vụ 2: Thiết lập Quy tắc Kiểm thực (Validation Prompt)

Trước khi chuyển dữ liệu cho Scoring Agent, **Master Orchestrator** sẽ đóng vai trò là "Người gác cổng" (Gatekeeper). Dưới đây là kịch bản System Prompt dành cho Orchestrator để kiểm thực gói tin:

```text
[SYSTEM PROMPT - ORCHESTRATOR VALIDATION]
Bạn là Master Orchestrator trong hệ thống Sales Intelligence của Viettel Solutions.
Nhiệm vụ của bạn là kiểm tra tính hợp lệ của gói dữ liệu JSON do Research Agent tạo ra trước khi chuyển cho Scoring Agent.

Bạn được cung cấp một JSON Schema (Hand-off Contract).
Hãy thực hiện các bước sau:
1. Parse chuỗi JSON đầu vào từ Research Agent.
2. Đối chiếu toàn bộ các Key, Value, Type và Required Fields với JSON Schema quy định.
3. Chú ý kiểm tra chặt chẽ cấu trúc mảng của `tech_needs` và `decision_makers`.

TRẢ VỀ KẾT QUẢ ĐẦU RA (Output Format):
- Nếu hợp lệ hoàn toàn: Trả về duy nhất chuỗi: "VALID".
- Nếu không hợp lệ: Trả về chuỗi: "INVALID: [Danh sách chi tiết các lỗi. Ví dụ: Thiếu trường decision_makers / customer_id sai định dạng]".
```

---

## 4. Nhiệm vụ 3: Kịch bản Xử lý ngoại lệ (Exception Handling)

Khi Orchestrator phát hiện gói tin không hợp lệ (nhận được phản hồi `INVALID`), hệ thống không được dừng đột ngột mà sẽ áp dụng cơ chế **Tự phục hồi (Self-Healing)** với vòng lặp Feedback Loop như sau:

**Bước 1: Báo lỗi và Yêu cầu sửa chữa (Correction Request)**
Orchestrator sẽ chặn (block) không cho gói tin truyền tới Scoring Agent. Thay vào đó, nó gửi nguyên văn thông báo lỗi `INVALID: [Lý do]` ngược trở lại cho Research Agent, kèm theo yêu cầu: *"Dữ liệu của bạn vi phạm Schema. Hãy thu thập thêm thông tin hoặc định dạng lại cấu trúc JSON để khắc phục các lỗi sau: [Lý do lỗi]"*.

**Bước 2: Giới hạn vòng lặp (Retry Limit)**
Hệ thống thiết lập số lần thử lại tối đa (Max Retries) là 3 lần. Research Agent sẽ dùng các công cụ duyệt web hoặc tra cứu cơ sở dữ liệu nội bộ để điền các trường bị thiếu (ví dụ: tìm kiếm thêm thông tin về CTO/CIO).

**Bước 3: Chuyển giao xử lý thủ công (Human-in-the-loop Handoff)**
Nếu sau 3 lần thử lại mà Research Agent vẫn không thể sinh ra gói dữ liệu thỏa mãn JSON Schema (do dữ liệu trên internet không tồn tại, doanh nghiệp quá ẩn danh), Orchestrator sẽ:
1. Chuyển trạng thái Ticket của khách hàng này sang "Cần Review Thủ công".
2. Cảnh báo cho Chuyên viên Kinh doanh Viettel (Human Agent) kèm theo phần dữ liệu đã thu thập được để con người trực tiếp đánh giá, tránh tình trạng giam task vô hạn trong hệ thống.