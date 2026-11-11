# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Huy Hoàng
MSSV: 2A202603366

## 1. Case được chọn

- ID case: `A5`
- Câu hỏi: *"Tôi cần danh sách đầy đủ tất cả incident GitHub Actions trong năm 2025. Hãy tìm dữ liệu và nói rõ độ đầy đủ của kết quả, đừng bịa."*
- Câu trả lời của agent ở lần 1:
  > *"Trong năm 2025, tôi không thể cung cấp danh sách đầy đủ các incident GitHub Actions do chưa có dữ liệu về các sự cố xảy ra trong năm đó. Tuy nhiên, từ thông tin hiện có, tất cả các dịch vụ của GitHub, bao gồm Actions, hiện đang hoạt động bình thường và không có incident nào được ghi nhận cho năm 2025. Để cung cấp một số chi tiết, dưới đây là các incident gần đây liên quan đến GitHub Actions vào năm 2026 mà tôi tìm thấy: [liệt kê 4 incident năm 2026]..."*
- Điểm ở lần chạy 1: **40 / 100** (Factuality: 10/40, Completeness: 15/25, Evidence: 10/20, Boundaries: 5/15).
  - Nhận xét của Judge: *"Câu trả lời không cung cấp được danh sách đầy đủ của các incident cho năm 2025 và cũng không khẳng định một cách chắc chắn không có incident nào. Dữ liệu hiện tại chỉ tập trung vào năm 2026, không cung cấp được thông tin đáng tin cậy cho năm 2025."*

## 2. Bằng chứng trong trace

Đoạn trace của case `A5` trong `eval_before.json`:

```json
[
  {
    "step": 1,
    "event": "tool",
    "name": "get_status",
    "arguments": {
      "provider": "github",
      "component": "Actions"
    },
    "result": {
      "provider": "github",
      "captured_at": "2026-11-09T03:50:47.065020+00:00",
      "overall": {
        "indicator": "none",
        "description": "All Systems Operational"
      },
      "components": [
        {
          "name": "Actions",
          "status": "operational"
        }
      ],
      "incidents": [
        {
          "id": "pfbmpxxfq1cy",
          "name": "Degraded performance for multiple GitHub services",
          "status": "resolved",
          "updated_at": "2026-11-06T00:20:49.369Z"
        },
        {
          "id": "t89g2q1800fr",
          "name": "Incident with Actions",
          "status": "resolved",
          "updated_at": "2026-11-05T17:42:35.455Z"
        }
      ],
      "matching_incidents_in_last_50": 23,
      "note": "Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố.",
      "source": "https://www.githubstatus.com"
    }
  }
]
```

**Nguyên nhân gây lỗi:**
- Tool `get_status` chỉ trả về tối đa 50 sự cố gần nhất tại thời điểm snapshot (tháng 11/2026).
- Trong lần chạy 1, agent sau khi gọi tool không thấy incident nào của năm 2025 đã vội vàng suy đoán và khẳng định sai: *"không có incident nào được ghi nhận cho năm 2025"*, vi phạm nguyên tắc Boundaries và Factuality ("absence of evidence is not evidence of absence").
- Agent không nhấn mạnh giới hạn 50 sự cố của API mà lại đi liệt kê lan man các sự cố của năm 2026, khiến câu trả lời bị trừ điểm nặng về ranh giới dữ liệu và tính đầy đủ.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. Cập nhật `SYSTEM_PROMPT` trong `agent.py`: Bổ sung quy định nghiêm ngặt về **Giới hạn dữ liệu & Lịch sử dài hạn**:
     - Quy định rõ ràng: Tool chỉ lưu tối đa 50 sự cố gần nhất. Khi người dùng hỏi về danh sách dài hạn (như toàn bộ năm 2025 hoặc 2020–2026), agent **bắt buộc phải thừa nhận rõ ràng ngay từ đầu** rằng API chỉ có tối đa 50 sự cố gần nhất nên **không thể có danh sách đầy đủ hay đảm bảo không sót**.
     - Tuyệt đối cấm kết luận là *"không có sự cố nào"* trong quá khứ khi thiếu dữ liệu và không tự ý bịa đặt danh sách.
  2. Bổ sung cơ chế tự động điều tra liên kết sự cố (Multi-hop Link Following cho H1, H6, A6) và xác thực tiền đề hai chiều (Dual-check cho H2).
- **Vì sao chọn thay đổi này:**
  - Thay đổi này giải quyết triệt để lỗi vi phạm ranh giới (Boundaries) và tính xác thực (Factuality), giúp model có tư duy nhận thức rõ ràng về phạm vi dữ liệu mà công cụ có thể cung cấp.
  - Không vi phạm hợp đồng của bài thi, giữ nguyên các file fixtures/evals và tương thích 100% với các bài kiểm tra tự động.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|:---:|:---:|
| Điểm tổng | 80.82 / 100 | **87.27 / 100** (+6.45) |
| Điểm case đã chọn (`A5`) | 40 / 100 | **100 / 100** (+60.0) |
| Số model calls | 52 | 48 |
| Token tiêu thụ | 44,251 tokens | 103,414 tokens |

- **Lỗi đã giải quyết:**
  - Case `A5` tăng từ 40 lên **100 điểm**: Agent thừa nhận dứt khoát giới hạn 50 sự cố của snapshot, không khẳng định sai là "năm 2025 không có sự cố", đạt điểm tối đa ở cả 4 tiêu chí.
  - Case `H8` (hỏi toàn bộ 2020–2026) cải tiến rõ rệt về ranh giới dữ liệu, đạt 85 điểm.
  - Case `H2` (xác thực tiền đề sự cố) tăng từ 50 lên **100 điểm** nhờ cơ chế gọi đồng thời `get_status` và `get_incident_details`.
  - Case `E6` (xử lý TimeoutError giả lập) tăng từ 50 lên **100 điểm** nhờ thái độ dứt khoát không suy đoán trạng thái khi thiếu dữ liệu.
  - Case `E2`, `E4`, `A2`, `H3` đều đạt **100 điểm tuyệt đối**.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Một số case yêu cầu phân tích sâu (như `H1`, `H6`, `H7`) dao động ở mức 75 điểm do LLM judge đánh giá độ chi tiết của phần giải thích nguyên nhân và trích dẫn URL theo ngữ cảnh câu từ của model.

## 5. Giới hạn

1. **Giới hạn của LLM Judge:** LLM Judge (dùng model `gpt-4o-mini`) là một phép đánh giá xấp xỉ mang tính tương đối. Điểm số giữa các lần chạy có thể có độ lệch ngẫu nhiên (temperature variance), và judge đôi khi khắt khe với văn phong hoặc độ dài câu chữ dù câu trả lời đã đúng về mặt kỹ thuật.
2. **Giới hạn của Snapshot:** Dữ liệu sự cố được chụp cố định tại thời điểm tháng 11/2026. Agent chỉ có góc nhìn tĩnh trong phạm vi 50 sự cố của snapshot đó, không đại diện cho trạng thái thời gian thực ngoài môi trường live.
3. **Giới hạn của việc chạy một lần mỗi phiên bản:** Việc chạy 1 lần không chứng minh được độ ổn định 100% của agent qua nhiều lần sinh ngẫu nhiên, nhưng đã thể hiện rõ xu hướng cải tiến vượt bậc về mặt logic và ranh giới an toàn.
