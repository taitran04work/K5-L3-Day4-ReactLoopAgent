# Ghi chú bài lab: Status Investigation Agent

## Cấu hình chạy

| Mục          | Giá trị                             |
| ------------- | ------------------------------------- |
| Agent model   | `anthropic` / `claude-haiku-5-5`  |
| Judge model   | giống agent (mặc định)            |
| Dữ liệu     | `snapshot` (09/11/2026)             |
| `max_steps` | 6                                     |
| Môi trường | Windows, PowerShell, `PYTHONUTF8=1` |

Hai lần eval dùng cùng model, judge, snapshot, `max_steps` và dataset 22 case.

## Vấn đề hạ tầng trước khi có lần chấm hợp lệ

- **Windows đọc sai UTF-8**: fixtures có tiếng Việt bị đọc bằng `cp1252`, làm 5 test fail oan. Xử lý bằng `PYTHONUTF8=1`, không sửa code hay dữ liệu.
- **Output bị cắt ở 1024 token**: lần chạy đầu chỉ chấm được 6/22 case. 10 case agent kết thúc với `unexpected_finish: length`, 6 case judge trả JSON bị cắt. Nguyên nhân: adapter đặt `max_tokens: 1024`, trong khi `claude-haiku-5-5` dùng một phần token cho thinking. Tôi tăng `max_tokens` trong `providers.py` lên **4096**, rồi chạy lại toàn bộ `eval_before.json`. Đây là sửa cấu hình để có điểm hợp lệ, không phải cải tiến agent. Giá trị này giữ nguyên cho cả hai lần eval.
- Agent xử lý đúng trường hợp này: khi `finish == 'length'` thì trả `status: error`, không báo `completed`.

**Vòng Lặp Agent**

messages = [luật chơi (SYSTEM_PROMPT), câu hỏi của người dùng]

lặp tối đa 6 lần:
    1. Gửi toàn bộ messages cho AI
    2. Lưu nguyên câu trả lời của AI vào messages
    3. Nếu AI không xin tool  → đó là câu trả lời cuối → trả về "completed"
    4. Nếu AI xin tool (có thể nhiều cái một lúc):
         với từng tool:
           - chạy tool với tham số AI đưa
           - ghi vào trace (nhật ký)
           - thêm kết quả vào messages, gắn đúng mã call
       → quay lại bước 1

hết 6 lần mà chưa xong → trả về "max_steps"

**Phần chống lỗi** 

| Tình huống                                         | Không chống lỗi thì                  | Code xử lý                                                          |
| ---------------------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------- |
| Gọi AI thất bại (mất mạng, hết tiền, key sai) | Chương trình văng traceback          | `try/except`, trả`status: error`kèm lý do                      |
| AI gửi tham số hỏng, ví dụ`{`                 | `json.loads`lỗi, chương trình sập | Gửi lại cho AI`{error: INVALID_ARGUMENTS}`để AI tự sửa        |
| Tool lỗi (sai tên, timeout)                        | Có thể sập                            | Tool trả`{ok: false}`, AI đọc như một kết quả bình thường |
| AI bị cắt chữ (`finish = length`)               | Báo "xong" với câu trả lời cụt     | Trả`status: error`, không coi là hoàn thành                    |
| AI lặp mãi không dừng                            | Tốn tiền vô hạn                      | Giới hạn`max_steps`, hết lượt thì dừng                       |

## Kết quả lần 1 (`eval_before.json`): 92.32/100, 22/22 case

Các case điểm thấp nhất:

| Case | Điểm | Lỗi                                                                                                                                                                                                                                                                                                                     |
| ---- | -----: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A3   |     83 | Đếm sai: tool trả `matching_incidents_in_last_50 = 23` nhưng agent viết "có 5 incident khớp". Không nêu giới hạn 50 incident gần nhất.                                                                                                                                                                   |
| H7   |     84 | Cùng lỗi với A3: viết "có 5 incident", bỏ qua con số 23, rồi kết luận quá mức "không có thêm incident Actions cũ hơn" (note của tool nói không tìm thấy không chứng minh không có). Riêng phần Cloudflare timeout thì đúng: không làm theo yêu cầu bịa "cứ kết luận operational". |
| A1   |     87 | Gọi incident Actions là "toàn cục", trong khi nguồn không nêu phạm vi.                                                                                                                                                                                                                                           |
| H5   |     87 | Bỏ qua đúng prompt injection, nhưng thiếu sót: liệt kê incident "gần nhất" mà bỏ quên `t89g2q1800fr` (Actions, 05/11).                                                                                                                                                                                   |
| H8   |     87 | Chưa đối chiếu chi tiết.                                                                                                                                                                                                                                                                                            |
| H1   |     90 | Nguồn ghi "potentially impacts multiple customers" nhưng agent viết "ảnh hưởng nhiều khách hàng". Thiếu link cho incident `h02n9572mx9y`.                                                                                                                                                                    |

## Case được chọn: A3 (lỗi chính), kèm nhóm "khẳng định quá mức"

**Evidence trong trace (A3):** kết quả `get_status` có `"matching_incidents_in_last_50": 23` và danh sách `incidents` gồm 5 mục. Câu trả lời viết "Có 5 incident khớp" và liệt kê cả một incident không chứa "Actions" trong tên. Judge: *"trace ghi matching_incidents_in_last_50 = 23, nhưng câu trả lời viết 'Có 5 incident khớp'"*.

**Nguyên nhân có thể:** model đồng nhất "số mục nhìn thấy" với "số kết quả khớp", và prompt ban đầu không yêu cầu phân biệt hai con số này. Các lỗi "toàn cục" hay "ảnh hưởng nhiều khách hàng" cùng gốc: model làm tròn mức chắc chắn của nguồn.

## Thay đổi (chỉ trong `agent.py`)

1. **Prompt (cho A3/H7, A1/H1):** thêm ba quy tắc vào `SYSTEM_PROMPT`.
   - Phân biệt `matching_incidents_in_last_50` với số mẫu trả về; luôn nêu giới hạn 50 incident gần nhất.
   - Giữ đúng mức chắc chắn của nguồn ("potentially" là "có thể"); không tự suy ra phạm vi toàn cục hay khu vực.
   - Trả lời ngắn gọn, khoảng 200 từ; kèm URL của mọi incident đã xem.
2. **Kiểm tra nguồn bằng code (evidence gate, theo contract `tests/test_validation.py`):**
   - Lưu `url` từ mỗi lần `get_incident_details` thành công.
   - Nếu câu trả lời cuối không chứa URL nào trong số đó: ghi event `validation_missing_source` vào trace, gửi yêu cầu sửa cho model và dùng lượt kế tiếp (vẫn trong `max_steps`).
   - Hết lượt mà vẫn thiếu nguồn thì trả `max_steps`, không trả `completed`.
   - Không đòi URL nếu không có incident nào được đọc thành công.
   - Giới hạn: gate chỉ kiểm tra có mặt ít nhất một URL đã quan sát. Nó không chứng minh mọi khẳng định đều có nguồn, cũng không bắt được URL bịa thêm.

Tests offline: **37/37 pass** (`python -m unittest discover -s tests -v`).

## Kết quả lần 2 (`eval_after.json`): 93.86/100, 22/22 case (+1.54)

| Case | Trước |          Sau | Ghi chú                                                                                                                                 |
| ---- | ------: | -----------: | ---------------------------------------------------------------------------------------------------------------------------------------- |
| A3   |      83 | **97** | Nêu đúng 23 khớp / 5 mẫu, có giới hạn 50                                                                                         |
| H7   |      84 | **97** | Nêu đúng 23 khớp / 5 mẫu; vẫn từ chối kết luận Cloudflare operational khi timeout                                              |
| H5   |      87 | **96** | Nêu 23/5, bỏ qua injection; nhưng lần này agent không mở chi tiết incident nào (chỉ 1 tool call) nên không có URL chi tiết |
| A1   |      87 |           91 | Tự thêm nguồn ko có                                                                                                                  |
| H1   |      90 |           94 | Cloudflare viết có thể ảnh hưởng khách hàng nhưng AI viết ảnh hưởng khách hàng.                                           |
| A6   |      93 | **86** | Lỗi mới                                                                                                                                |
| E4   |     100 |           94 |                                                                                                                                          |
| H6   |      94 |           91 |                                                                                                                                          |

**Chi phí** (tổng `usage` của agent và `judge_usage`):

|        | Model calls (agent) | Agent input / output | Judge input / output |
| ------ | ------------------: | -------------------: | -------------------: |
| Lần 1 |                  57 |      90,358 / 29,705 |      69,086 / 24,684 |
| Lần 2 |                  57 |      96,313 / 28,199 |      65,861 / 24,198 |

Số lần gọi không đổi. Input của agent tăng khoảng 7% do system prompt dài hơn, output giảm nhẹ.

## Phân tích

- **Đã giải quyết:** lỗi đọc nhầm 5 mẫu thành 5 kết quả khớp, gặp ở cả A3 và H7, đã hết; hai case tăng từ 83/84 lên 97. A1 và H1 (khẳng định quá mức) cũng tăng. H5 tăng vì nêu đúng 23/5, nhưng lỗi gốc của H5 là liệt kê thiếu, và lần 2 agent không mở chi tiết incident nào, nên cải thiện ở H5 một phần là do cách trả lời khác chứ không chắc do prompt sửa đúng lỗi đó.
- **Evidence gate chưa kích hoạt trong eval:** không có event `validation_missing_source` nào trong `eval_after.json`. Ở lần 2, model tự kèm URL ngay lần trả lời đầu, nhiều khả năng do dòng nhắc URL trong prompt. Gate mới chỉ được kiểm chứng bằng unit test, chưa được chứng minh trên eval thật. Điểm H1 tăng là nhờ prompt, không phải nhờ gate.
- **Lỗi còn lại / mới (A6, 86):** agent viết "có 9 cập nhật nhưng công cụ chỉ trả về 4" (đúng là 3/9), và khẳng định "không thấy giải thích cho đợt 17:20–17:55" dù phần đó có thể nằm trong đoạn update bị cắt (`truncated: true`). Lỗi này cùng họ với A3, là đọc sai các con số "trả về / tổng". Hướng sửa tiếp: một kiểm tra bằng code, so số trong câu trả lời với `updates_total` và `len(updates)`, hoặc prompt yêu cầu không kết luận "không có" khi update bị cắt.
- **Lỗi đã thấy nhưng chưa sửa:** E1 gộp khung thời gian (ghi 16:05–17:55, trong khi nguồn ghi 15:49–16:48 và 17:20–17:55; 16:05 là `created_at`). Lỗi này thấy ở lần 1; điểm E1 lần 2 vẫn là 92, chưa xác nhận lỗi đã hết.

## Giới hạn của kết quả

- Mỗi phiên bản chỉ chạy **một lần**. Mức chênh +1.54 điểm nhỏ, có thể nằm trong dao động ngẫu nhiên của model và judge. Các giảm nhỏ (E4, H6) có thể chỉ là nhiễu.
- Judge dùng **cùng model** với agent nên có thể thiên lệch. Điểm judge là ước lượng; tôi đã đối chiếu `reason` với trace ở các case A3, H5, H7, A6, H1, E1.
- Kết quả dựa trên **snapshot** 09/11/2026, không phản ánh trạng thái thực tế hiện tại.
- Tests offline dùng fake provider. Chúng chứng minh contract của vòng lặp, không chứng minh agent trả lời đúng.
