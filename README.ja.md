<h1 align="center">guardrail-chatbot-jev</h1>

<p align="center">
  <b>AI チャットボットのためのコンテンツ安全レイヤー。ひとつのポリシーファイルでユーザーの送信内容、ボットの返答、会話全体を検査し、判定ごとに深刻度、取るべき対応、根拠となる標準を返します。</b><br>
  MLCommons AILuminate、Llama Guard、OWASP に対応づけた 18 の危険カテゴリ · Python、TypeScript、Go で同じ判定、サードパーティ依存なし · 実環境のマルチターン評価で無害な追質問の保留 0 / 194、有害な返答の検出 19 / 19 · ベトナム語、英語、中国語の定型返答を備えたベトナム準拠パック
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.vi.md">Tiếng Việt</a> · <a href="README.fr.md">Français</a> · <b>日本語</b>
  <br>
  <a href="https://nhatnguyet.org/tro-ly-ai">ライブデモ</a> ·
  <a href="docs/guide.ja.md">ガイド</a> ·
  <a href="docs/multiturn.ja.md">マルチターンの手法</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.md">ベトナム準拠</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases">リリース</a>
</p>

<p align="center">
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml/badge.svg"></a>
  <a href="LICENSE"><img alt="License: CC BY-NC 4.0" src="https://img.shields.io/badge/license-CC%20BY--NC%204.0-lightgrey.svg"></a>
  <img alt="Python 3.10+" src="https://img.shields.io/badge/python-3.10%2B-blue.svg">
  <img alt="Node 20+" src="https://img.shields.io/badge/node-20%2B-brightgreen.svg">
  <img alt="Go 1.22+" src="https://img.shields.io/badge/go-1.22%2B-00ADD8.svg">
</p>

## クイックスタート

```bash
pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"
```

以下はキーもネットワークも不要でそのまま動きます。トランスポートが Jev モデルの返す形式の回答を再生します。

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

本番ではトランスポートを外し、`JEV_API_KEY` を設定します。統合はモデルの前と後に一度ずつ検査するだけです。

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
<summary>TypeScript と Go</summary>

TypeScript（Node 20+）は [`ts/`](ts/) にあります：`cd ts && npm install && npm run build`。

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

Go（[`go-sdk`](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/go-sdk/go/README.md) ブランチ）。モジュールパスは公開時の名前のままです。

```bash
go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4
```

```go
guard := guardrail.New(guardrail.Options{})
verdict, err := guard.CheckInput(ctx, userMessage, nil)
```

</details>

## 判定が伝えること

検査はそれぞれ二つの独立した答えを返します。`action` は内容の深刻度、`route` はそれをどう扱うかです。漏れた電話番号と爆弾の作り方はどちらも深刻になり得ますが、前者はマスクして送り、後者は保留します。ひとつのスコアではこれを表せません。

| `action` | 意味 | | `route` | コード側の対応 |
| --- | --- | --- | --- | --- |
| `allow` | 何も検出されない | | `deliver` | そのまま送る |
| `flag` | 記録するが止めない | | `redact` | 機微な部分をマスクして送る |
| `review` | 確認せずに通さない | | `guide` | 方向を変えた回答を送る |
| `block` | 保留する | | `crisis_support` | 危機支援の情報で返答する |
| | | | `human_review` | 人の確認待ちに入れる |
| | | | `safe_response` | 用意した定型の拒否を送る |

各検出結果はカテゴリ、確率、そして `refs`（対応する標準、ベトナム準拠パックでは法律）を持つため、保留した理由を監査人に説明できます。Jev に接続できないとき、判定は `degraded: true` となり内容については何も言いません。これは別に数えてください。

## 何を検査するか

| 検査 | 読む対象 | 検出するもの |
| --- | --- | --- |
| `check_input` | ユーザーのメッセージ単体 | 有害な依頼、プロンプトインジェクション、個人データ |
| `check_output` | ボットの返答 | 有害または情報を漏らす返答、危険な助言、根拠のない主張 |
| `check_conversation` | 直近 10 件のメッセージ | 丁寧な複数ターンに分散したエスカレーション、ペルソナの逸脱 |

18 のカテゴリ、そのしきい値、それらをつなぐ 11 の規則はひとつのファイル [`policies/standard-v1.json`](policies/standard-v1.json) にあり、すべての SDK がこれを読みます。カテゴリの説明文はそのまま Jev への質問になるため、ポリシーを編集すればコードに触れずに挙動が変わります。一回の検査の質問はすべて一つのリクエストで送られ（入力面では 10 問）、内容は意味で判断されます。ポリシーは英語、ベトナム語、フランス語、日本語向けに書かれており、翻訳された依頼に甘くしないようモデルに指示しています。定義付きの全一覧：[危険カテゴリ一覧](skill/guardrail-chatbot-jev/references/taxonomy.md)。

検査の背後にあるのは [TypeSafe AI](https://typesafe.ai) の意思決定モデル **Jev** で、生成モデルではありません。ポリシーが定義したラベル上の較正済み確率を返すので、カテゴリを作り出したり内容について文章を書いたりできません。一回の検査は一往復で、通常 70〜500 ms です。

## 次のメッセージを罰さないマルチターン

会話を読むガードレールによくある失敗は *文脈汚染* です。履歴に違反が一度入ると、「ごめん、今のは忘れて。明日の天気は？」も含め、以降のすべてのターンがそれを引き継いでしまいます。本プロジェクトの規則はひとつです：**ターンを保留するのは、そのターン自体かその返答に根拠があるときだけ。** 履歴は返答をどれだけ注意深く読むかを決めるだけで、保留するかどうかは決めません。最近リスクのあった会話では、返答を以前のターンと合わせてもう一度読み、その読みは返答が以前の有害な依頼の続きを提供する場合にのみ数えます。

Jev 上の実測、5 言語・ラベル付き 223 会話（[`examples/multiturn-live.jsonl`](examples/multiturn-live.jsonl)）：

| | 以前の設計（セッション下限） | 現在（帰属判定） |
| --- | --- | --- |
| 保留された無害なターン | 171 / 194 | **0 / 194** |
| 検出された有害な返答 | 19 / 19 | **19 / 19** |
| フラグされたエスカレーション | 9 / 9 | **9 / 9** |
| レビュー待ちになった無害な会話 | 172 / 194 | **2 / 194** |

保留されたメッセージはプレースホルダーとしてセッションに残り、チャットモデルにはどのカテゴリで保留されたかだけが伝えられ、本文は決して渡されません。そのため「やって」のような追質問にも文脈に沿って答えられます。手法、すべての設定としきい値、数式、アブレーション、リプレイ結果は [docs/multiturn.ja.md](docs/multiturn.ja.md) にあります。シミュレーションのシナリオはオフラインで再実行できます：

```bash
cd ts && npm test
```

```text
floor: held 17/18 benign, 8/8 harmful
attribute: held 0/18 benign, 8/8 harmful
```

## ベトナム準拠パック

ベトナムの AI サービス向けに、`vietnam-compliance-v1` ポリシーは 18 カテゴリに 5 つのグループを加えます。各グループは根拠となる法律を示し、モデルに書かせる代わりにベトナム語、英語、中国語の定型返答で応じます。

| グループ | 根拠 |
| --- | --- |
| `vsv` 領土主権 | サイバーセキュリティ法 |
| `vas` 国家に反する宣伝 | サイバーセキュリティ法 |
| `vld` 国家指導者と国の象徴への侮辱 | サイバーセキュリティ法 |
| `vcs` 虚偽情報と公共秩序の攪乱 | サイバーセキュリティ法 |
| `vai` AI の欺瞞的・操作的な利用 | 人工知能法 |

普通の質問（天気、指導者の経歴、組織の問い合わせ窓口）はブロックしないよう調整されています。範囲、透明性、Python と Go での統合、較正、人による監督、記録の保持を扱う手順ガイド：
[English](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.md) ·
[Tiếng Việt](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.vi.md) ·
[中文](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.zh.md)。
[Nhat Nguyet AI アシスタント](https://nhatnguyet.org/tro-ly-ai)の裏側で本番稼働しています。

## 何が違うか

- **スコアではなく二つの軸。** 深刻度（`action`）と扱い（`route`）を別々に決めるため、リスクのある内容の多くは拒否ではなく、マスク、方向転換、支援情報の提示で処理されます。
- **追跡できる検出結果。** カテゴリは公開標準に由来し、各検出結果は対応する標準を名指しします。
- **ひとつのポリシー、三つの言語。** Python、TypeScript、Go が同じファイルを読み、同じ判定に至ります。三者は 40,000 件のランダムな回答セットで一致します。ある言語で保存したセッションは別の言語で復元できます。
- **マルチターンの帰属判定**。セッション全体にかける下限ではなく、上記のとおり計測済み。
- **サービスではなくライブラリ。** 判定を返し、決めるのはあなたのコードです。知らないところで何かがブロックされることはなく、サードパーティのパッケージも入りません。

隣接する課題を解く別のツールもあり、そちらが合う場合もあります：

| 必要なもの | 検討候補 |
| --- | --- |
| 自前でホストする安全性分類器 | Llama Guard |
| プログラム可能な対話フローとトピックのガードレール | NeMo Guardrails |
| 構造化出力や形式のためのバリデーター | Guardrails AI |
| ローカルで動く入力・出力スキャナー | LLM Guard |
| プロンプトインジェクションを選別するホスト型 API | Lakera Guard |

## 本番運用の準備

以下はすべてパッケージに含まれます。詳細、コード、レイテンシ表は [docs/production.ja.md](docs/production.ja.md) と[ガイド](docs/guide.ja.md)にあります。

- **入力検査をモデル呼び出しの前ではなく横で走らせる**ので、レイテンシはほとんど増えません。
- **ストリーミング**は検査の一つ後ろでテキストを出し、待つのは最初のチャンクだけです。
- **判定キャッシュと決定的プレフィルター**が、繰り返しや明白なケースをネットワーク呼び出しなしで処理します。
- **面ごとの障害時動作**：入力はオープン、出力はクローズ、常に `degraded` を付与。
- **セッション**は `as_state()` / `from_state()` で永続化できます。[`examples/session_store.py`](examples/session_store.py) にプロセス内と Redis のストアがあります。
- **リアルタイムモード**（`review_handling="audit"`）：内容を止めるのは `block` のみで、すべての判定に監査レベルが付きます。
- **独自のコンプライアンス領域**を、フォークではなく同梱ポリシーへのオーバーレイとして追加：[`examples/domain_policy.py`](examples/domain_policy.py)。
- **CLI** は終了コードで判定を返し、`--dry-run` でキーなしに送信内容をそのまま表示します。
- **オフライン調整**：実行を一度記録し、[`scripts/sweep.py`](scripts/sweep.py) で新しいしきい値に対して再生します。

## サンプル

どれも API キーなしで動き、記録済みの回答にフォールバックします。

| | |
| --- | --- |
| [`integration.py`](examples/integration.py) · [`integration.ts`](examples/integration.ts) | 保護された一ターンを端から端まで：並列の入力検査、ストリーミング、キャッシュ、プレフィルター、セッション、4 言語での同じ質問 |
| [`chatbot_server.py`](examples/chatbot_server.py) | 同じターンを FastAPI の背後で、SSE ストリーミングと会話ごとのセッション付きで |
| [`domain_policy.py`](examples/domain_policy.py) | 同梱ポリシーに独自のコンプライアンス領域を加える |

[`examples/`](examples/) には、較正用のラベル付きセット（入力 36 件、出力 15 件、会話 7 件）と、223 会話のマルチターンセットもあります。

## 限界

- **強制レイヤーではありません。** 判定を返すだけで、動くのはあなたのデプロイです。
- **Jev は与えられたものしか読みません。** 調べものや計算はできず、否定表現が弱点です。検索、レート制限、アカウント状態は周囲のコードの役割です。
- **同梱のしきい値は出発点であって測定値ではありません。** 自分のラベル付きトラフィックで較正してください。方法は[ガイド](docs/guide.ja.md)と [`scripts/`](scripts/) にあります。
- ラベル付きセットの違反テキストは 30 件で、児童安全の実際の陽性例はないため、そのカテゴリの再現率は未測定です。

## リリース

| リリース | 内容 | インストール |
| --- | --- | --- |
| [Python SDK 1.1.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python/v1.1.4) | Python と TypeScript、CLI、調整ツール | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"` |
| [Go SDK 1.2.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go/v1.2.4) | 同じエンジンの Go 版、実環境・リプレイ・回帰テストのツール付き | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4` |
| [Python ベトナム準拠パック 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python-vietnam-compliance-v1.2.3) | 3 言語の返答を備えた `vietnam-compliance-v1` | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python-vietnam-compliance-v1.2.3#subdirectory=python"` |
| [Go ベトナム準拠パック 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go-vietnam-compliance-v1.2.3) | 同じポリシーと返答の Go 版（モジュール `v1.3.3`） | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.3.3` |

Python と TypeScript は `main`、Go は `go-sdk`、ベトナム準拠パックは `guardrail-vietnam-compliance`（Python）と `go-vietnam-compliance`（Go）にあります。変更履歴は [CHANGELOG.md](CHANGELOG.md) にあります。

## コントリビュート

Issue とプルリクエストを歓迎します。[CONTRIBUTING.md](CONTRIBUTING.md) を参照してください。どちらのテストもキーやネットワークなしで動きます：

```bash
cd python && python -m pytest -q     # 101 tests
cd ts && npm test                    # 83 tests
```

`scripts/verify-all.sh` はポリシーパック、ラベル付きセット、CLI、すべてのサンプル、パッケージングも検証します。脆弱性の報告は [SECURITY.md](SECURITY.md) を参照してください。

## ライセンス

[CC BY-NC 4.0](LICENSE)：非営利目的であれば、出典を示して利用、共有、改変できます。商用利用には著作権者 Nhat Nguyet の個別の許可が必要です。

## 引用

```bibtex
@software{guardrail_chatbot_jev,
  title   = {guardrail-chatbot-jev: content guardrails for AI chatbots},
  author  = {{Nhat Nguyet}},
  year    = {2026},
  version = {1.1.4},
  url     = {https://github.com/Nhat-Nguyet/guardrail-chatbot-jev}
}
```
