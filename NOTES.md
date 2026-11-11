# Ghi chú phân tích · Day 04

Họ và tên: Nguyen Quoc Viet

MSSV: 2A202603556

## 1. Case được chọn

- ID case: A6
- Câu trả lời của agent: xác nhận Actions operational, tóm tắt hai incident GitHub và phạm vi/thời gian ảnh hưởng.
- Điểm: 100 ở lần trước và 100 ở lần sau.

## 2. Bằng chứng trong trace

Case A6 yêu cầu dẫn nguồn cho hai incident. Khi tắt kiểm tra nguồn trong lần baseline, trace ghi:

```text
validation_disabled, step 3
https://stspg.io/7pwv28hvkmz8
https://stspg.io/pjgptg3cfc6n
```

Ở lần sau, trace ghi `validation_missing_source` tại step 3 cho cùng hai URL. Agent gọi model thêm một lần; câu trả lời cuối có cả hai URL incident từ tool. Điểm case vẫn 100, nhưng lần sau hoàn thành yêu cầu dẫn đúng URL ngắn từ dữ liệu tool.

## 3. Thay đổi đã thực hiện

- Hoàn thiện vòng lặp tool trong `agent.py`: giữ nguyên assistant message, xử lý mọi tool call, trả kết quả với đúng `tool_call_id`, xử lý JSON sai, lỗi tool, giới hạn bước và ghi trace.
- Thêm kiểm tra URL incident trong `agent.py`. Khi tool trả incident thành công mà câu trả lời thiếu URL đã quan sát, agent yêu cầu model bổ sung trong ngân sách bước. Kiểm tra chỉ xác nhận URL hiện diện, không xác minh mọi nhận định.
- Thêm adapter tùy chọn xKiro trong `providers.py`, dùng OpenAI SDK với proxy môi trường tắt theo cấu hình của xKiro. Adapter cũng xử lý `tool_calls: null`. Cập nhật lựa chọn provider trong `run.py` và `evals/run_evals.py`.
- Baseline chạy với `STATUS_VALIDATE_SOURCES=0`; lần sau chạy với `STATUS_VALIDATE_SOURCES=1`.

## 4. So sánh hai lần chạy

Cả hai lần dùng xKiro `mistralai/mistral-medium-3.5` cho agent và judge, snapshot, dataset core, `max_steps=8`, 4 workers. Cả 22 case đều được chấm.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 95.0/100 | 95.0/100 |
| Điểm A6 | 100 | 100 |
| Agent model calls | 56 | 60 |
| Agent tokens | 62,214 | 81,034 |
| Judge tokens | 53,984 | 54,742 |
| Chi phí | Provider không trả trường chi phí | Provider không trả trường chi phí |

- Lỗi được xử lý: khi incident URL bị bỏ sót, trace ghi sự kiện kiểm tra và agent yêu cầu model sửa câu trả lời. Trong A6, lượt sửa đã đưa hai URL vào câu trả lời cuối.
- Lỗi còn lại: H7 vẫn khẳng định Cloudflare Spectrum operational sau khi tool Cloudflare báo timeout. Agent cần giữ trạng thái đó là chưa xác minh và nêu lỗi một phần. H6 còn 60 điểm; điểm judge và phần giải thích của judge có dấu hiệu không nhất quán, nên cần đối chiếu expected và trace thay vì chỉ dựa vào điểm.
- Điểm tổng không đổi; lượt kiểm tra nguồn làm tăng 4 agent calls và khoảng 30% agent tokens trong lần chạy này.

## 5. Giới hạn

Judge dùng cùng model với agent nên có thể thiên lệch hoặc giải thích không khớp điểm số. Snapshot được chụp ngày 2026-11-09, không thể xác nhận trạng thái hiện tại. Mỗi phiên bản chỉ chạy một lần nên chênh lệch nhỏ có thể do dao động model. Kiểm tra nguồn chỉ buộc URL incident đã quan sát xuất hiện trong câu trả lời; nó không chứng minh các claim có liên quan hoặc chính xác. H7 cho thấy timeout vẫn có thể bị model diễn giải sai thành trạng thái operational.
