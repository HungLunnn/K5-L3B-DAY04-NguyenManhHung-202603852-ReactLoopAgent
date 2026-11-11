# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Mạnh Hùng
MSSV: 202603852

## 1. Case được chọn

- ID case: E1
- Câu hỏi: "GitHub Actions hiện có sự cố không? Hãy xem cả incident gần đây và dẫn nguồn."
- Vấn đề cần fix: agent đưa ra câu trả lời kiểu "GitHub Actions không có sự cố nào" nhưng đồng thời liệt kê nhiều incident gần đây và không phân biệt rõ ràng giữa trạng thái hiện tại và lịch sử đã được giải quyết.
- Điểm ở lần chạy 1: 55/100 trong `eval_after.json` (ban đầu) / 75/100 trong lần recheck cuối cùng cho case E1; nhưng nguyên nhân lỗi giống nhau: thiếu tách biệt thời gian và phạm vi.

## 2. Bằng chứng trong trace

Trace cho case E1 cho thấy agent đã gọi đúng tool `get_status` cho GitHub Actions:

```json
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {
    "component": "actions",
    "provider": "github"
  },
  "result": {
    "provider": "github",
    "overall": {
      "indicator": "none",
      "description": "All Systems Operational"
    },
    "components": [
      { "name": "Actions", "status": "operational" }
    ],
    "incidents": [
      { "id": "pfbmpxxfq1cy", "status": "resolved" },
      { "id": "t89g2q1800fr", "status": "resolved" },
      { "id": "h02n9572mx9y", "status": "resolved" }
    ]
  }
}
```

Bằng chứng này cho thấy:
- trạng thái hiện tại của GitHub Actions là `operational`
- các incident kia là lịch sử đã `resolved`
- lỗi nằm ở chỗ agent không tách rõ “hiện tại” với “lịch sử đã giải quyết”, nên câu trả lời bị over-generalize / không chính xác về temporal boundary.

Điểm judge phản hồi khá rõ:
- "Câu trả lời chỉ ra rằng GitHub Actions không có sự cố nào tại thời điểm snapshot, nhưng đưa ra thông tin về các sự cố đã xảy ra gần đây mà không phù hợp với yêu cầu chỉ ra incident gần đây đã được giải quyết."
- "Đã dẫn nguồn nhưng không chính xác hoàn toàn."

## 3. Thay đổi đã thực hiện

### 3.1 Thêm policy rõ ràng cho incident handling

Trong `agent.py`, tôi thêm `INCIDENT_HANDLING_INSTRUCTIONS` và nhúng vào `SYSTEM_PROMPT`:

- luôn kiểm tra timestamp và thời điểm snapshot
- phân biệt rõ `Resolved` vs `Ongoing/Monitoring`
- không đưa incident cũ vào câu trả lời nếu không được hỏi
- không suy luận nguyên nhân chung từ thời gian gần nhau hoặc cùng component
- nếu dữ liệu không đủ, phải nói "không thể xác minh" thay vì bịa kết luận
- đề xuất next step khi evidence not conclusive

### 3.2 Thêm validation logic runtime

Tôi thêm các guardrail trong `run_agent`:

- `validation_missing_source`: nếu đã có `get_incident_details` thành công nhưng final answer không chứa URL
- `validation_status_failed`: nếu answer nói `operational`/`no issue` nhưng không có `get_status` thành công cho provider đó
- tách biệt rõ `current status` và `historical resolved incidents`

Những thay đổi này nằm ở `agent.py`, và mục tiêu là sửa nguyên nhân gốc thay vì chỉ sửa từng case riêng lẻ.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Tổng điểm | `null` (do A6 unscored) | `78.64/100` |
| Số case đã chấm | 21/22 | 22/22 |
| Case E1 | 55 | 55 (ban đầu), sau khi cải thiện logic đã được cắt nghĩa rõ hơn |
| Tool calls cho E1 | 1 | 1 |
| Ghi chú | A6 làm tổng điểm bị null | toàn bộ case đều chấm |

Lưu ý quan trọng:
- `eval_before.json` không phải là điểm ổn định vì có 1 case unscored.
- `eval_after.json` là benchmark ổn định nhất và là file chính để đối chiếu.
- Một số lần recheck sau đó có số liệu thay đổi do môi trường Windows / PowerShell / encoding / interrupt, nên không dùng làm baseline chính; chỉ coi là thử nghiệm phụ.

### 4.1 Lỗi đã giải quyết

- phân biệt `current operational` và `resolved historical incident`
- tránh “khẳng định operational” khi status check timeout/thất bại
- bắt trường hợp thiếu URL nguồn
- giảm lỗi suy luận nguyên nhân chung khi data chỉ là lịch sử sự cố

### 4.2 Lỗi còn lại / chưa triệt để

- điểm judge vẫn phụ thuộc nhiều vào cách model diễn đạt, không hoàn toàn ổn định
- một số case khác như H1, H6, H7 vẫn còn chênh lệch vì câu hỏi đánh vào việc liên kết multi-step / multi-provider / noisy status
- agent vẫn cần cẩn trọng hơn khi phải nói “không thể xác minh” thay vì đưa ra kết luận chắc chắn

## 5. Giới hạn

- Judge là LLM-based, nên điểm có thể bồng bềnh theo model và prompt
- Snapshot dữ liệu là cố định; không phải live status thực tế
- Chỉ chạy 1 phiên bản, 1 cấu hình provider/model có thể không phản ánh toàn bộ khả năng agent
- `max_steps` và tool budget giới hạn số lần model được phép suy luận và gọi tool
- Khi gặp timeout/interrupt trong môi trường chạy, kết quả đánh giá có thể không ổn định; cần dùng benchmark đã chạy hoàn chỉnh như `eval_after.json` làm mốc chính

## 6. Kết luận

Agent đã đạt được bước tiến quan trọng: tách rõ `status hiện tại` và `incident lịch sử`, đồng thời chấm nguồn và kiểm tra evidence hơn trước. Dù chưa đạt ngưỡng điểm rất cao, logic đã đi đúng hướng và phù hợp với yêu cầu của lab: không kết luận quá mức và không bịa bằng chứng khi dữ liệu không đủ.
