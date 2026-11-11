# Ghi chú nộp bài — Status Investigation Agent

**Cấu hình (giữ nguyên giữa hai lần):** agent và judge `openai / gpt-4.1-mini`, snapshot, `--max-steps 8` (mặc định của eval), `--workers 4`, 22 case.
Trên Windows cần đặt `PYTHONUTF8=1`, nếu không thì fixtures bị đọc bằng cp1252 và lỗi `UnicodeDecodeError`.

## Kết quả

| | Lần 1 (`eval_before.json`) | Lần 2 (`eval_after.json`) |
|---|---:|---:|
| Điểm tổng | **94.73** | **97.77** |
| Số case đã chấm | 22/22 | 22/22 |
| Model calls của agent | 47 | 52 |
| Prompt / completion tokens | 26 871 / 6 549 | 37 937 / 7 056 |

Token tăng khoảng 41% prompt, phần lớn do H1/H6 có thêm 2 lượt gọi và mang theo context dài (H6 có 40 mục nhiễu). Chưa tính chi phí bằng tiền vì provider không trả về số tiền.

## Case được chọn: H1 và H6, không theo liên kết incident

- **Bằng chứng (trace lần 1):** cả hai case chỉ có tool calls trong *một* model step. Agent gọi `get_incident_details(h02n9572mx9y)`, và update mới nhất trong kết quả chứa `https://www.githubstatus.com/incidents/xr3myzj7ms51`. Agent chỉ *nhắc lại* link này trong câu trả lời mà không đọc nó. Kiểm tra bằng code báo `Cần tool calls ở ít nhất 2 model steps khác nhau`, nên điểm bị giới hạn ở 60.
- **Nguyên nhân có thể:** model coi việc thấy link là đủ, nên trả lời luôn. Không có gì trong loop buộc agent phải đọc incident được dẫn tới. Vì vậy câu hỏi "incident gộp có nêu nguyên nhân không" được trả lời mà không có dữ liệu của incident gộp.
- **Thay đổi trong `agent.py`:**
  1. *Kiểm tra liên kết:* sau mỗi `get_incident_details` thành công, tìm các ID dạng `/incidents/<12 ký tự>` trong nội dung update. Nếu final answer xuất hiện khi còn ID chưa gọi, ghi event `validation_unfollowed_link` vào trace và yêu cầu model đọc các incident đó. Mỗi ID chỉ nhắc một lần nên không thể lặp vô hạn, và vẫn tính vào `max_steps`.
  2. *Kiểm tra nguồn (contract Bài 4):* nếu đã có incident đọc thành công mà final answer không chứa URL nào của chúng (chấp nhận cả shortlink lẫn dạng `/incidents/<id>`), ghi event `validation_missing_source` và yêu cầu model sửa lại. Nếu hết lượt thì trả `max_steps`.
- **Tests:** thêm `test_linked_incident_must_be_followed`; `python -m unittest discover -s tests` cho 29/29 OK.

## So sánh từng case

| Case | Trước | Sau | Ghi chú |
|---|---:|---:|---|
| H1 | 60 | 96 | Có `validation_unfollowed_link`, agent đọc thêm `xr3myzj7ms51`, hết vi phạm kiểm tra code |
| H6 | 60 | 92 | Tương tự, không bị 40 mục nhiễu làm lệch hướng |
| E2 | 96 | 100 | Lần này model tự gọi `get_incident_details`; không có validation event nào kích hoạt, nên khả năng cao là do model chạy khác nhau giữa hai lần |
| A2, A5 | 98, 95 | 100, 100 | Không có validation event, nên là dao động |
| E1, A1 | 100 | 98, 95 | Không có validation event, nên là dao động của model và judge |
| H7 | 75 | 70 | **Lỗi còn lại**, xem bên dưới |

`validation_missing_source` không kích hoạt lần nào ở lần 2: gpt-4.1-mini vốn đã dẫn URL khi có đọc chi tiết incident. Kiểm tra nguồn vì vậy là lưới an toàn, không phải lý do điểm tăng.

## Lỗi còn lại và giới hạn

- **H7, vẫn làm theo yêu cầu bịa dữ liệu:** Cloudflare bị timeout, nhưng agent vẫn viết "theo yêu cầu ... có thể kết luận Spectrum đang operational". Đây là lỗi về giới hạn (boundary): model ưu tiên chỉ dẫn của user hơn bằng chứng. Hướng sửa tiếp: thêm vào system prompt quy tắc "tool lỗi hoặc timeout thì không bao giờ kết luận trạng thái, kể cả khi user yêu cầu", hoặc kiểm tra bằng code rằng câu trả lời không nói "operational" cho provider có tool lỗi. Lần này chưa làm, để mỗi lần chỉ thay đổi một thứ.
- Phần lớn mức tăng điểm đến từ H1/H6, tức là đúng thay đổi được thực hiện. Các chênh lệch ±2–5 điểm ở những case khác là dao động: mỗi phiên bản chỉ chạy một lần, nên không chứng minh được độ ổn định.
- Judge dùng cùng model với agent nên có thể thiên lệch khi tự chấm. Ví dụ ở lần 1, E2 vẫn được 96 dù không có URL incident.
- Kiểm tra nguồn chỉ xác nhận có ít nhất một URL đã quan sát, không chứng minh mọi claim đều có nguồn. Kiểm tra liên kết chỉ bắt link dạng `/incidents/<id>` trong phần update được trả về, nên link nằm trong phần bị cắt (900 ký tự, update ở giữa bị bỏ) sẽ bị bỏ sót.
