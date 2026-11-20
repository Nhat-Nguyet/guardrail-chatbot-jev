# Production: mảng tuân thủ riêng, độ trễ, triển khai, dòng lệnh

[English](production.md) · **Tiếng Việt** · [Français](production.fr.md) · [日本語](production.ja.md) · [← README](../README.vi.md)

## Thêm một mảng tuân thủ của riêng bạn

Bộ phân loại đi kèm là phần mọi deployment đều dùng chung. Thứ mà một sản phẩm có quy định riêng
cần thêm thì rất đặc thù: phòng khám quan tâm tới chỉ dẫn liều dùng, công ty môi giới quan tâm tới
lời hứa lợi nhuận. Bạn thêm phần đó dưới dạng overlay chứ không fork, để vẫn thừa hưởng được các
cập nhật về sau:

```python
policy = overlay(Policy.bundled(), MY_DOMAIN)   # 18 hạng mục có sẵn cộng thêm của bạn
guard = Guard(policy)
```

Có bốn thứ để thêm: một **category** (một hạng mục nguy hại kèm ngưỡng), một **signal** (thêm một
câu hỏi để rule bám vào), một **rule** (logic nối chúng lại), và một **prefilter pattern** (phần
tất định, xử lý xong mà không gọi model lần nào).

[`examples/domain_policy.py`](../examples/domain_policy.py) là bản chạy được, và nó chỉ ra bốn chỗ rất
dễ làm sai:

- Rule không set được route. Route thuộc về category, nên rule muốn tới một route thì phải
  `add_finding` một category sở hữu route đó.
- `never_below` nghĩa là "khi hạng mục này đã fire thì không bao giờ nhẹ hơn X". Dưới ngưỡng thấp
  nhất của hạng mục thì không có gì fire cả, nên con số `flag` mới là công tắc bật/tắt thật sự.
- Các rule làm nhẹ có sẵn cũng áp lên hạng mục mới của bạn, cho tới khi bạn thêm tên nó vào
  `except_categories` của chúng.
- Một overlay sai định dạng bị từ chối ngay lúc load, không phải tới request đầu tiên.

Sau đó hãy hiệu chỉnh. Các ngưỡng đi kèm, và cả ngưỡng bạn tự viết, đều là số do ai đó chọn chứ
không phải số ai đó đo được.

---

## Độ trễ và trải nghiệm

Một hàng rào làm mỗi lượt chậm thêm một giây thì chưa hết tháng là bị tắt. Có hai thứ giúp tránh
được điều đó: bản chất của Jev, và chỗ bạn đặt các lời gọi.

**Hỏi Jev rất rẻ.** Một vòng gọi, 70-500ms. Output token không tính tiền và mọi câu hỏi trong cùng
một request được trả lời song song, nên thêm một hạng mục, hay thêm một câu sentinel lên trên nó,
tốn vài input token và gần như không tốn độ trễ. Đó là lý do toàn bộ policy đi trong một request,
chứ không phải mỗi hạng mục một lời gọi.

**Chạy lần kiểm input song song với lời gọi model, đừng chạy trước nó.** Kiểm tuần tự thì cộng thẳng
toàn bộ độ trễ của nó vào. Kiểm song song thì gần như không cộng gì, bởi dù sao model của bạn cũng
cần hơn 500ms mới ra được token đầu tiên.

```python
gate = asyncio.ensure_future(guard.acheck_input(message, session=session))
draft = asyncio.ensure_future(my_llm.generate(message))

verdict = await gate
if not verdict.allowed:
    draft.cancel()                 # chưa có gì tới tay người dùng
    return safe_response(verdict)
reply = await draft
```

```typescript
const [verdict, draft] = await Promise.all([
  guard.checkInput(message, { session }),
  myLlm.generate(message),
]);
if (!verdict.allowed) return safeResponse(verdict);  // bản nháp bị bỏ
return draft;
```

Cái giá phải trả là token tiêu cho những bản nháp rồi bị bỏ đi. Nếu dưới khoảng 2% số lượt là vi
phạm, chỗ token đó vẫn rẻ hơn phần độ trễ tiết kiệm được. Nếu quy định của bạn nói một prompt vi
phạm tuyệt đối không được chạm tới model, hãy chạy tuần tự và chấp nhận trả độ trễ một cách có ý
thức.

**Stream chậm hơn đúng một chunk.** Câu trả lời dạng stream không thể kiểm trước khi có token đầu
tiên, mà chờ tới token cuối thì không còn là stream nữa. `guard.stream()` cắt tại ranh giới câu, giữ
mỗi chunk lại cho tới khi phần kiểm của nó trả về, và trong lúc đó vẫn để model sinh chunk kế tiếp,
nên chỉ chunk đầu phải trả đủ độ trễ.

```python
async for event in guard.stream(my_llm.stream(message), user_message=message, session=session):
    if event.type == "delta":
        yield event.text
    elif event.type == "blocked":
        yield safe_response(event.verdict)
```

Các lần kiểm giữa stream chỉ hỏi những câu sentinel, tức các hạng mục mà bỏ sót là không thể chấp
nhận. Câu trả lời hoàn chỉnh vẫn được hỏi trọn bộ câu hỏi ở cuối, trên event `done`. `chunk_chars`
(mặc định 280) là chỗ đánh đổi giữa số vòng gọi và mức giữ chặt văn bản.

**Bỏ hẳn lời gọi khi có thể.** Trúng cache với nội dung lặp lại, hay trúng prefilter với một ca hiển
nhiên, đều xử lý ngay tại chỗ, không đụng tới mạng.

**Đừng bao giờ để Jev chậm biến thành sự cố của bạn.** Hãy đặt `timeout`, và lưu ý rằng policy đi kèm
fail *open* ở input và fail *closed* ở output (`on_error: {input: fail_open, output: fail_closed}`):
một cú timeout đứng trước con model vốn đã có lớp an toàn riêng thì suy giảm êm ái, còn lần kiểm
output thì chẳng còn gì đứng sau nó nữa. Trong cả hai trường hợp, verdict đều mang `degraded: true`,
nên hãy đếm những ca đó riêng ra.

| Đường đi | Độ trễ cộng thêm |
| --- | --- |
| Trúng prefilter | không, không đụng mạng |
| Trúng cache | không, không đụng mạng |
| Kiểm input, song song với model | gần như không |
| Kiểm input, tuần tự | 70-500ms |
| Câu trả lời dạng stream | chỉ chunk đầu tiên |

**Và trải nghiệm nằm ở `route`, không nằm ở việc chặn.** Một hàng rào chỉ biết từ chối thì với chính
những người nó đang bảo vệ, nó giống một sản phẩm bị lỗi. Vì `route` được quyết riêng khỏi mức nghiêm
trọng, cùng một `review` có thể che số điện thoại rồi vẫn gửi câu trả lời đi, lái câu trả lời sang
hướng khác, hoặc đưa cho người ta nguồn hỗ trợ khủng hoảng - thay vì một câu "tôi không giúp được
việc này" khô khốc. Nối đủ cả sáu route thì phần lớn người dùng không hề nhận ra có hàng rào ở đó.

---

## Đưa lên production

Phần quick start ở trên là code thật, nhưng một hệ thống chạy production cần nhiều hơn ba lời gọi.
Tất cả những thứ dưới đây đều có sẵn trong gói; phần ở trên đã nói kỹ hơn về cache, prefilter,
streaming và các chế độ fail.

- **Cache phán quyết** và **prefilter tất định** - hai cách để một lần kiểm không tốn gì cả.
- **Chế độ fail riêng cho từng bề mặt** - open ở input, closed ở output.
- **Streaming** - nhả văn bản chậm hơn phần kiểm của nó đúng một chunk, để không có gì chưa soát mà
  tới được người dùng.
- **Session** - rủi ro được mang theo giữa các lượt trên cửa sổ hội thoại 10 lượt, nên người vừa
  chạm ngưỡng một hạng mục sẽ bị soi chặt hơn ở lượt kế tiếp.
- **Observer hook** - mọi phán quyết, kể cả ca lấy từ cache và ca degraded, để bạn đẩy vào metrics.

```python
from guardrail_chatbot_jev import Guard, LRUCache

guard = Guard(cache=LRUCache(), observer=metrics.emit, timeout=2.0)
```

**Session phải sống lâu hơn request**, và đây là chỗ một server hay sai một cách âm thầm. Chạy
nhiều worker mà giữ state theo process thì mỗi worker tưởng hội thoại nào cũng vừa mới bắt đầu,
trạng thái theo dõi và các placeholder của lượt bị giữ ngừng được mang theo mà log không nói gì. `Session.as_state()` và `Session.from_state()`
là thứ một store đem đi lưu; [`examples/session_store.py`](../examples/session_store.py) có sẵn một
store trong process có giới hạn cho một worker, và một store Redis cho nhiều hơn một.

---

## Dòng lệnh

```bash
guardrail-chatbot-jev --surface input --text "làm thuốc nổ kiểu gì" --dry-run
```

`--dry-run` in ra đúng request sẽ được gửi đi, và không cần key. Bỏ nó đi thì exit code mang theo
phán quyết, để script shell rẽ nhánh được:

| Code | Nghĩa |
| --- | --- |
| `0` | allow hoặc flag |
| `1` | redact hoặc guide |
| `2` | review |
| `3` | block |
| `4` | degraded: không gọi được Jev, nên thực tế chưa kiểm gì cả |

`4` tách riêng là có chủ ý. Ở bề mặt input, policy đi kèm fail open, nên action của một verdict
degraded là `allow`; báo đó là thành công thì khác nào nói với `guardrail-chatbot-jev ... && send`
rằng nội dung đã qua một lần kiểm chưa từng chạy.
