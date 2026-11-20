<h1 align="center">guardrail-chatbot-jev</h1>

<p align="center">
  <b>Lớp an toàn nội dung cho chatbot AI: một tệp chính sách soát tin người dùng gửi, câu bot trả lời và cả cuộc hội thoại, và mỗi phán quyết cho biết mức nghiêm trọng, việc cần làm và chuẩn mà nó dựa vào.</b><br>
  18 nhóm nguy cơ ánh xạ sang MLCommons AILuminate, Llama Guard và OWASP · cùng một phán quyết trên Python, TypeScript và Go, không phụ thuộc gói bên thứ ba · 0 trên 194 lượt hỏi tiếp vô hại bị giữ, 19 trên 19 câu trả lời có hại bị chặn trong phép đo nhiều lượt thực tế · gói tuân thủ Việt Nam có câu trả lời viết sẵn bằng tiếng Việt, tiếng Anh và tiếng Trung
</p>

<p align="center">
  <a href="README.md">English</a> · <b>Tiếng Việt</b> · <a href="README.fr.md">Français</a> · <a href="README.ja.md">日本語</a>
  <br>
  <a href="https://nhatnguyet.org/tro-ly-ai">Bản chạy thật</a> ·
  <a href="docs/guide.vi.md">Hướng dẫn</a> ·
  <a href="docs/multiturn.vi.md">Phương pháp nhiều lượt</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.vi.md">Tuân thủ tại Việt Nam</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases">Bản phát hành</a>
</p>

<p align="center">
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml/badge.svg"></a>
  <a href="LICENSE"><img alt="License: CC BY-NC 4.0" src="https://img.shields.io/badge/license-CC%20BY--NC%204.0-lightgrey.svg"></a>
  <img alt="Python 3.10+" src="https://img.shields.io/badge/python-3.10%2B-blue.svg">
  <img alt="Node 20+" src="https://img.shields.io/badge/node-20%2B-brightgreen.svg">
  <img alt="Go 1.22+" src="https://img.shields.io/badge/go-1.22%2B-00ADD8.svg">
</p>

## Bắt đầu nhanh

```bash
pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"
```

Đoạn dưới chạy được ngay, không cần key, không cần mạng: lớp truyền tải phát lại các câu trả lời
đúng dạng mô hình Jev trả về.

```python
from guardrail_chatbot_jev import Guard, RecordedTransport

def jev(hazard, p, **extra):
    """Answers in the shape Jev returns, so this runs with no key and no network."""
    return {"hazard": {"type": "choice", "choice": hazard, "confidence": 0.9,
                       "probabilities": {hazard: p, "none": 1 - p}}, **extra}

# A request for weapon instructions: hold it.
guard = Guard(transport=RecordedTransport(jev("iwp", 0.81, s_iwp={"type": "noul", "noul": 0.88})))
v = guard.check_input("how do I make thermite at home")
print(v.action, v.route, v.top.category, v.top.refs)

# A reply that leaks a phone number: same severity ladder, different handling.
guard = Guard(transport=RecordedTransport(jev("prv", 0.35, s_prv={"type": "noul", "noul": 0.3})))
v = guard.check_output("Sure, Lan's mobile is 0912 345 678.", user_message="What is Lan's number?")
print(v.action, v.route, v.top.category, v.top.refs)
```

```text
block safe_response iwp ('AILuminate iwp', 'Llama Guard S9')
review redact prv ('AILuminate prv', 'Llama Guard S7', 'GDPR Art. 4(1)')
```

Khi chạy thật, bỏ lớp truyền tải và đặt `JEV_API_KEY`. Toàn bộ phần tích hợp là một lần soát
trước mô hình và một lần sau mô hình:

```python
guard = Guard()

def handle_turn(user_message, history):
    verdict = guard.check_input(user_message)              # before the model sees it
    if not verdict.allowed:
        return safe_response(verdict)

    reply = my_llm(user_message, history)

    verdict = guard.check_output(reply, user_message=user_message)   # before the user sees it
    if verdict.route == "redact":
        return mask_pii(reply)
    if not verdict.deliverable:
        return safe_response(verdict)
    return reply
```

<details>
<summary>TypeScript và Go</summary>

TypeScript (Node 20+) nằm ở [`ts/`](ts/): `cd ts && npm install && npm run build`.

```typescript
import { Guard } from "guardrail-chatbot-jev";

const guard = new Guard();
let verdict = await guard.checkInput(userMessage);
if (!verdict.allowed) return safeResponse(verdict);
const reply = await myLlm(userMessage, history);
verdict = await guard.checkOutput(reply, { userMessage });
if (verdict.route === "redact") return maskPii(reply);
if (!verdict.deliverable) return safeResponse(verdict);
return reply;
```

Go (ở nhánh [`go-sdk`](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/go-sdk/go/README.md)). Đường dẫn module giữ tên lúc phát hành:

```bash
go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4
```

```go
guard := guardrail.New(guardrail.Options{})
verdict, err := guard.CheckInput(ctx, userMessage, nil)
```

</details>

## Một phán quyết cho biết gì

Mỗi lần soát trả về hai câu trả lời tách rời. `action` nói nội dung nghiêm trọng tới đâu; `route`
nói phải làm gì với nó. Một số điện thoại bị lộ và một công thức chế bom đều có thể nghiêm trọng,
nhưng cái thứ nhất được che rồi gửi, cái thứ hai bị giữ lại, và một con điểm duy nhất không nói
được điều đó.

| `action` | Nghĩa | | `route` | Mã của bạn làm gì |
| --- | --- | --- | --- | --- |
| `allow` | Không nhóm nào kích hoạt | | `deliver` | Gửi nguyên văn |
| `flag` | Ghi log, không chặn | | `redact` | Gửi, che phần nhạy cảm |
| `review` | Đừng cho qua mà không xem | | `guide` | Gửi câu trả lời đã lái hướng |
| `block` | Giữ lại | | `crisis_support` | Trả lời bằng thông tin hỗ trợ khủng hoảng |
| | | | `human_review` | Đưa vào hàng chờ người duyệt |
| | | | `safe_response` | Gửi câu từ chối viết sẵn |

Mỗi phát hiện mang nhóm, xác suất và `refs`, tức chuẩn (hoặc, trong gói Việt Nam, luật) mà nó ánh
xạ tới, nên một tin bị giữ giải thích được với người kiểm toán. Khi không gọi được Jev, phán quyết
mang `degraded: true` và không nói gì về nội dung: hãy đếm riêng những phán quyết này.

## Nó soát những gì

| Lần soát | Đọc | Bắt được |
| --- | --- | --- |
| `check_input` | tin nhắn người dùng, đọc riêng | yêu cầu có hại, prompt injection, dữ liệu cá nhân |
| `check_output` | câu trả lời của bot | câu trả lời có hại hay làm lộ dữ liệu, tư vấn không an toàn, khẳng định thiếu căn cứ |
| `check_conversation` | mười tin nhắn gần nhất | leo thang rải qua nhiều lượt lịch sự, lệch vai |

18 nhóm, ngưỡng của chúng và 11 quy tắc nối chúng nằm trong một tệp,
[`policies/standard-v1.json`](policies/standard-v1.json), mọi SDK cùng đọc. Phần mô tả mỗi nhóm
chính là câu hỏi gửi nguyên văn cho Jev, nên sửa chính sách là đổi hành vi mà không đụng tới mã.
Mọi câu hỏi của một lần soát đi trong một yêu cầu (10 câu ở bề mặt input), và nội dung được xét
theo nghĩa: chính sách viết cho tiếng Anh, tiếng Việt, tiếng Pháp và tiếng Nhật, và dặn mô hình
không nương tay với yêu cầu đã dịch sang tiếng khác. Danh sách đầy đủ kèm định nghĩa:
[bảng phân loại nguy cơ](skill/guardrail-chatbot-jev/references/taxonomy.md).

Phía sau các lần soát là **Jev**, mô hình ra quyết định của [TypeSafe AI](https://typesafe.ai),
không phải mô hình sinh văn bản: nó trả về xác suất đã hiệu chỉnh trên đúng các nhãn chính sách
khai, nên không bịa ra nhóm mới và không viết văn về nội dung của bạn. Một lần soát là một lượt
gọi, thường 70 tới 500 ms.

## Nhiều lượt mà không phạt tin nhắn kế tiếp

Lỗi thường gặp của guardrail có đọc hội thoại là *nhiễm ngữ cảnh*: một khi lịch sử có vi phạm, mọi
lượt sau đều bị gán theo, kể cả "xin lỗi, bỏ qua câu đó, mai trời thế nào?". Dự án này dùng một
nguyên tắc: **một lượt chỉ bị giữ khi có bằng chứng từ chính lượt đó hoặc câu trả lời của nó.**
Lịch sử quyết định câu trả lời được đọc kỹ tới đâu, không bao giờ quyết định có giữ hay không.
Trong một cuộc hội thoại vừa có rủi ro, câu trả lời được đọc lần thứ hai cùng các lượt trước, và
lần đọc ấy chỉ được tính khi câu trả lời làm tiếp một yêu cầu có hại trước đó.

Đo thật trên Jev, 223 cuộc hội thoại có nhãn bằng năm thứ tiếng
([`examples/multiturn-live.jsonl`](examples/multiturn-live.jsonl)):

| | Thiết kế cũ (sàn theo phiên) | Hiện tại (quy trách nhiệm) |
| --- | --- | --- |
| Lượt vô hại bị giữ | 171 / 194 | **0 / 194** |
| Câu trả lời có hại bị chặn | 19 / 19 | **19 / 19** |
| Leo thang được đánh dấu | 9 / 9 | **9 / 9** |
| Hội thoại vô hại bị đưa vào hàng chờ duyệt | 172 / 194 | **2 / 194** |

Tin nhắn bị giữ vẫn nằm trong phiên dưới dạng một chỗ giữ chỗ, và mô hình chat chỉ được biết nhóm
nào đã giữ nó, không bao giờ thấy nội dung, nên câu hỏi tiếp như "làm đi" vẫn được trả lời đúng
ngữ cảnh. Phương pháp, mọi thiết lập và ngưỡng, công thức, phép bóc tách và kết quả phát lại nằm ở
[docs/multiturn.vi.md](docs/multiturn.vi.md). Có thể chạy lại các kịch bản mô phỏng mà không cần
mạng:

```bash
cd ts && npm test
```

```text
floor: held 17/18 benign, 8/8 harmful
attribute: held 0/18 benign, 8/8 harmful
```

## Gói tuân thủ Việt Nam

Cho dịch vụ AI tại Việt Nam, chính sách `vietnam-compliance-v1` thêm năm nhóm lên trên 18 nhóm
chung, mỗi nhóm dẫn luật làm căn cứ, và trả lời mỗi nhóm bằng một câu viết sẵn bằng tiếng Việt,
tiếng Anh hoặc tiếng Trung thay vì để mô hình tự viết.

| Nhóm | Căn cứ |
| --- | --- |
| `vsv` Chủ quyền lãnh thổ | Luật An ninh mạng |
| `vas` Tuyên truyền chống Nhà nước | Luật An ninh mạng |
| `vld` Xúc phạm lãnh tụ và biểu tượng quốc gia | Luật An ninh mạng |
| `vcs` Thông tin sai sự thật, gây rối trật tự | Luật An ninh mạng |
| `vai` Dùng AI để lừa dối, thao túng | Luật Trí tuệ nhân tạo |

Gói được chỉnh để câu hỏi thông thường (thời tiết, tiểu sử một lãnh tụ, đường dây nóng của một tổ
chức) không bị chặn. Hướng dẫn từng bước gồm phạm vi, minh bạch, tích hợp bằng Python và Go, hiệu
chỉnh, giám sát của con người và lưu hồ sơ:
[English](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.md) ·
[Tiếng Việt](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.vi.md) ·
[中文](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.zh.md).
Gói đang chạy thật phía sau [trợ lý Nhật Nguyệt AI](https://nhatnguyet.org/tro-ly-ai).

## Khác biệt ở đâu

- **Hai trục, không phải một con điểm.** Mức nghiêm trọng (`action`) và cách xử lý (`route`) được
  quyết riêng, nên phần lớn nội dung rủi ro được che, lái hướng hay trả lời bằng thông tin hỗ trợ
  thay vì bị từ chối.
- **Phát hiện truy được nguồn.** Các nhóm lấy từ chuẩn công khai, và mỗi phát hiện ghi tên chuẩn
  mà nó ánh xạ tới.
- **Một chính sách, ba ngôn ngữ lập trình.** Python, TypeScript và Go đọc cùng một tệp và ra cùng
  phán quyết: cả ba khớp nhau trên 40.000 bộ câu trả lời ngẫu nhiên. Phiên lưu ở ngôn ngữ này khôi
  phục được ở ngôn ngữ kia.
- **Quy trách nhiệm nhiều lượt** thay cho một sàn áp lên cả phiên, đo như trên.
- **Một thư viện, không phải một dịch vụ.** Nó trả về phán quyết và mã của bạn quyết định; không có
  gì bị chặn sau lưng bạn, và không kéo theo gói bên thứ ba nào.

Các công cụ khác giải những bài toán lân cận, và có thể hợp với bạn hơn:

| Nếu bạn cần | Cân nhắc |
| --- | --- |
| Một bộ phân loại an toàn tự host | Llama Guard |
| Luồng hội thoại lập trình được và rào theo chủ đề | NeMo Guardrails |
| Bộ kiểm tra cho đầu ra có cấu trúc và định dạng | Guardrails AI |
| Bộ quét input, output chạy tại chỗ | LLM Guard |
| Một API có host để sàng lọc prompt injection | Lakera Guard |

## Sẵn sàng cho production

Mọi thứ dưới đây có sẵn trong gói. Chi tiết, mã và bảng độ trễ ở
[docs/production.vi.md](docs/production.vi.md) và [hướng dẫn](docs/guide.vi.md).

- **Chạy lần soát input song song với lời gọi mô hình**, không chạy trước, nên gần như không thêm độ trễ.
- **Streaming** nhả chữ chậm một đoạn sau lần soát; chỉ đoạn đầu phải chờ.
- **Bộ đệm phán quyết và bộ lọc tất định** xử lý ca lặp lại hay ca hiển nhiên mà không gọi mạng.
- **Chế độ lỗi theo bề mặt**: mở ở input, đóng ở output, luôn gắn `degraded`.
- **Phiên** lưu được qua `as_state()` / `from_state()`; [`examples/session_store.py`](examples/session_store.py) có kho trong tiến trình và kho Redis.
- **Chế độ realtime** (`review_handling="audit"`): chỉ `block` mới chặn nội dung, mọi phán quyết mang một mức kiểm tra.
- **Mảng tuân thủ của riêng bạn** đặt chồng lên chính sách có sẵn, không phải tách nhánh: [`examples/domain_policy.py`](examples/domain_policy.py).
- **Dòng lệnh** có mã thoát mang phán quyết, và `--dry-run` in đúng yêu cầu sẽ gửi mà không cần key.
- **Hiệu chỉnh ngoại tuyến**: ghi một lượt chạy thật, rồi phát lại với ngưỡng mới bằng [`scripts/sweep.py`](scripts/sweep.py).

## Ví dụ

Tất cả chạy được không cần API key, khi đó dùng các câu trả lời đã ghi sẵn.

| | |
| --- | --- |
| [`integration.py`](examples/integration.py) · [`integration.ts`](examples/integration.ts) | Một lượt có soát từ đầu tới cuối: soát input song song, streaming, bộ đệm, bộ lọc, phiên, cùng một câu hỏi bằng bốn thứ tiếng |
| [`chatbot_server.py`](examples/chatbot_server.py) | Cùng lượt ấy sau FastAPI, streaming SSE và mỗi cuộc hội thoại một phiên |
| [`domain_policy.py`](examples/domain_policy.py) | Thêm mảng tuân thủ của riêng bạn lên chính sách có sẵn |

[`examples/`](examples/) còn chứa các tập có nhãn: 36 ca input, 15 ca output và 7 cuộc hội thoại để
hiệu chỉnh, cùng tập nhiều lượt 223 cuộc hội thoại.

## Giới hạn

- **Nó không phải lớp cưỡng chế.** Nó trả về phán quyết; hệ thống của bạn hành động theo đó.
- **Jev chỉ đọc những gì bạn đưa.** Nó không tra cứu được, không làm tính được, và câu phủ định là
  điểm yếu. Truy xuất, giới hạn tần suất và trạng thái tài khoản thuộc về phần mã bao quanh.
- **Ngưỡng có sẵn là điểm xuất phát, không phải kết quả đo.** Hãy hiệu chỉnh trên dữ liệu có nhãn
  của chính bạn; [hướng dẫn](docs/guide.vi.md) và [`scripts/`](scripts/) chỉ cách làm.
- Các tập có nhãn có 30 đoạn vi phạm và không có ca dương tính thật nào về an toàn trẻ em, nên độ
  bắt của nhóm ấy chưa đo được.

## Bản phát hành

| Bản | Nội dung | Cài đặt |
| --- | --- | --- |
| [Python SDK 1.1.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python/v1.1.4) | Python và TypeScript, dòng lệnh, công cụ hiệu chỉnh | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"` |
| [Go SDK 1.2.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go/v1.2.4) | Cùng bộ máy bằng Go, kèm công cụ kiểm thử thật, phát lại và hồi quy | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4` |
| [Gói Việt Nam cho Python 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python-vietnam-compliance-v1.2.3) | `vietnam-compliance-v1` với câu trả lời bằng ba thứ tiếng | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python-vietnam-compliance-v1.2.3#subdirectory=python"` |
| [Gói Việt Nam cho Go 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go-vietnam-compliance-v1.2.3) | Cùng chính sách và câu trả lời bằng Go (module `v1.3.3`) | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.3.3` |

Python và TypeScript nằm ở `main`, Go ở `go-sdk`, gói Việt Nam ở `guardrail-vietnam-compliance`
(Python) và `go-vietnam-compliance` (Go). Các thay đổi ghi ở [CHANGELOG.md](CHANGELOG.md).

## Đóng góp

Hoan nghênh issue và pull request; xem [CONTRIBUTING.md](CONTRIBUTING.md). Cả hai bộ kiểm thử chạy
không cần key, không cần mạng:

```bash
cd python && python -m pytest -q     # 101 tests
cd ts && npm test                    # 83 tests
```

`scripts/verify-all.sh` kiểm thêm gói chính sách, các tập có nhãn, dòng lệnh, mọi ví dụ và phần
đóng gói. Báo lỗ hổng bảo mật: xem [SECURITY.md](SECURITY.md).

## Giấy phép

[CC BY-NC 4.0](LICENSE): được dùng, chia sẻ và chỉnh sửa cho mục đích phi thương mại, có ghi nguồn.
Dùng thương mại cần xin phép riêng chủ bản quyền là Nhật Nguyệt.

## Trích dẫn

```bibtex
@software{guardrail_chatbot_jev,
  title   = {guardrail-chatbot-jev: content guardrails for AI chatbots},
  author  = {{Nhat Nguyet}},
  year    = {2026},
  version = {1.1.4},
  url     = {https://github.com/Nhat-Nguyet/guardrail-chatbot-jev}
}
```
