# Ghi chú phân tích · Day 04

Họ và tên: Dang Khac Hoang Minh  
MSSV: 2A202604082

## 1. Case được chọn

- **ID:** H1
- **Yêu cầu:** Điều tra incident GitHub `h02n9572mx9y`, theo liên kết đến incident gộp để tìm nguyên nhân và ảnh hưởng, rồi so sánh với Cloudflare `yaa0witbl2me`. Nêu thời gian, trạng thái, component, nguồn của từng bên và không suy luận hai incident có chung nguyên nhân.
- **Lỗi ở lần chạy đầu:** Agent gọi hai `get_incident_details` trong cùng một lượt model. Vì không đọc nội dung incident GitHub trước khi gọi Cloudflare, agent bỏ qua incident GitHub gộp `xr3myzj7ms51`, nơi có nguyên nhân và ảnh hưởng chi tiết. Kiểm tra cơ học yêu cầu tool calls ở ít nhất hai lượt model, nên điểm H1 bị giới hạn ở 60.
- **Điểm:** 60/100 trong `eval_before.json`; 98/100 trong `eval_after.json` đang lưu.

## 2. Bằng chứng trong trace

Trace của `eval_before.json` cho thấy cả hai lời gọi diễn ra ở step 1:

```text
step 1: get_incident_details(provider="github", incident_id="h02n9572mx9y")
step 1: get_incident_details(provider="cloudflare", incident_id="yaa0witbl2me")
Mechanical check: cần tool calls ở ít nhất 2 model steps khác nhau; score_cap=60.
```

Trong `eval_after.json`, agent đọc incident GitHub gốc, đọc incident liên kết ở lượt tiếp theo, rồi mới truy vấn incident Cloudflare:

```text
step 1: get_incident_details(provider="github", incident_id="h02n9572mx9y")
step 2: get_incident_details(provider="github", incident_id="xr3myzj7ms51")
step 3: get_incident_details(provider="cloudflare", incident_id="yaa0witbl2me")
Mechanical check: không có vi phạm; 3 tool steps.
```

## 3. Thay đổi đã thực hiện

- Trong [agent.py](/E:/K5-L3B-Day4-DangKhacHoangMinh-2A202604082-ReactLoopAgent/agent.py), hướng dẫn agent điều tra tuần tự khi câu trả lời phụ thuộc vào kết quả trước đó: đọc incident được liên kết trước khi tổng hợp và kiểm tra status riêng trước khi đọc chi tiết khi câu hỏi yêu cầu cả hai.
- Thêm kiểm tra cho khẳng định status không có căn cứ sau timeout, giới hạn suy luận từ số incident được trả về và xử lý URL nguồn thiếu.
- Hoàn thiện hỗ trợ bonus: dùng schema/system prompt do tools cung cấp; giới hạn retry cho cùng request; đọc `Retry-After` và chỉ chờ tối đa 2 giây; giữ lỗi để model quan sát; theo dõi trang incident qua `next_offset`; không cho agent tuyên bố đã đọc hết timeline khi còn trang; lược bỏ danh sách `distractors` khỏi context và ghi số mục đã lược trong trace.
- Lý do chọn các thay đổi: H1 ban đầu gộp các tool calls và bỏ qua incident được liên kết. Các case timeout, lịch sử incident, pagination và dữ liệu nhiễu cũng cần phân biệt rõ bằng chứng đã có với phần chưa kiểm tra.

## 4. So sánh các lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` đang lưu |
|---|---:|---:|
| Bộ dữ liệu / cấu hình | 22 case, OpenAI `gpt-4.1-mini`, tối đa 8 bước | Cùng cấu hình |
| Điểm tổng | 90.50/100 | 95.32/100 |
| H1 | 60 | 98 |
| H2 | 50 | 50 |
| H6 | 60 | 99 |
| H7 | 50 | 85 |
| Model calls của agent | 49 | 48 |
| Token agent | 34,412 | 43,812 |
| Token judge | 33,209 | 31,875 |
| Tổng token được báo cáo | 67,621 | 75,687 |
| Chi phí | Không có trong file kết quả | Không có trong file kết quả |

Điểm tổng trong file tăng **4.82 điểm**. H1 và H6 cải thiện rõ sau khi agent điều tra nhiều bước. Tổng token tăng dù số model calls giảm nhẹ; các lần gọi dài hơn có thể là một nguyên nhân, nhưng một lần chạy không đủ để kết luận nguyên nhân hay độ ổn định.

Lỗi còn lại trong bài chính: H2 vẫn 50 vì agent chỉ đọc chi tiết incident `t89g2q1800fr`, chưa gọi `get_status` để kiểm tra tình trạng GitHub Actions trong snapshot. H7 đạt 85; agent không bịa trạng thái Cloudflare sau timeout nhưng phần completeness vẫn bị judge trừ điểm. A5 giảm từ 93 xuống 90; judge có thể dao động giữa các lần chạy và cần đối chiếu câu trả lời/trace thay vì chỉ nhìn điểm.

## 5. Kết quả bonus và giới hạn còn lại

File `eval_bonus.json` đang lưu ghi nhận **90/100**, tương đương **13.5/15**; đủ 10/10 case được chấm, cấu hình `gpt-4.1-mini`, tối đa 16 bước.

- **B6 = 50:** agent gọi status Vercel và npm nhưng không gọi `get_incident_details`; kiểm tra cơ học áp score cap 50.
- **B10 = 50:** Vercel trả `RATE_LIMITED`, nhưng trace chỉ có một lần gọi `get_vercel_status`; agent không retry trong case đó, dù code giới hạn retry và hướng dẫn model xử lý rate limit.
- Vì vậy, giới hạn retry trong agent ngăn lặp vô hạn nhưng không đảm bảo model sẽ chủ động retry. Tương tự, pagination vẫn cần model nhận ra `next_offset` liên quan đến câu hỏi và gọi trang tiếp theo.
- Kết quả bonus là một lần chạy với snapshot; chưa chứng minh agent sẽ xử lý ổn định mọi trường hợp live, mọi `Retry-After`, hoặc timeline dài.

## 6. Giới hạn của kết quả

- LLM judge chỉ là ước lượng, có thể sai hoặc dao động. Judge trong các file này dùng cùng model `gpt-4.1-mini` với agent, nên có thể có thiên lệch chung.
- Snapshot GitHub/Cloudflare và các fixture bonus là dữ liệu cố định tại thời điểm lưu. Kết quả không đại diện cho trạng thái hiện tại của các dịch vụ.
- Mỗi phiên bản được chạy một lần. Chênh lệch điểm không chứng minh mức cải thiện ổn định; cần xem expected, answer, trace và kiểm tra cơ học.
- Điểm trong ghi chú lấy từ các JSON hiện có trên đĩa. Console output đã gửi trước đó cho `eval_after.json` ghi 92.59/100, trong khi file đang lưu ghi 95.32/100. Hãy giữ đúng file kết quả tương ứng với lần chạy muốn nộp và đảm bảo `NOTES.md` khớp với file đó.
