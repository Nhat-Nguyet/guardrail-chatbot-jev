# マルチターン：手法、設定、評価

[English](multiturn.md) · [Tiếng Việt](multiturn.vi.md) · [Français](multiturn.fr.md) · **日本語** · [← README](../README.ja.md)


## 問題

**文脈汚染：** 分類器が履歴中の違反を読むと、次のターンの内容にかかわらず同じ違反を割り当てる。
比較基準（セッションのフロア、以前の設計）：実測で無害な後続 **194 件中 171 件**を保留。

## 設計原則

ターンを保留する根拠は、そのターン自身またはその応答から得た証拠に限る。履歴は読む深さを決めるだけで、
保留の可否は決めない。

## 手法

各ターンは、次の三つの問いに順に答えて判定します。

1. **ユーザーのメッセージは、それ自体で有害か？** 直前までの会話を付けずに単独で読みます。有害ならここで
   止めます。それ自体が無害なメッセージが、過去の発言を理由に止められることはありません。
2. **応答は、それ自体で有害か？** アシスタントの応答も同じく単独で読みます。
3. **会話に最近リスクがあった場合のみ：応答が、以前の有害な依頼を完成させていないか？** 応答を以前のターンと
   一緒にもう一度読みます。この二度目の読みが効くのは、応答が以前の有害な依頼の次の手順、詳細、翻訳、
   言い換えを与える場合だけです。謝罪、法律の質問、通報の方法、話題の変更なら効きません。

止めたメッセージは `[earlier message omitted]` として会話に残ります。試みがあったことは覚えていますが、
本文は二度と読みません。モデルには、メッセージが止められたことと止めたカテゴリ名だけを伝え、本文は見せません。
そのため「やって」「最初の依頼」といった続きの発言にも、推測ではなく文脈に沿って答えられます。理由はセッションの保存状態に残ります。

| 手順 | コード |
| --- | --- |
| 1. ユーザーのメッセージを単独で読む | [`check_input`](../python/src/guardrail_chatbot_jev/guard.py#L119) |
| 2. 応答を単独で読む | [`check_output`](../python/src/guardrail_chatbot_jev/guard.py#L131) |
| 3. 応答を以前のターンと一緒に読み、効くかを判断する | [`_check_in_context`](../python/src/guardrail_chatbot_jev/guard.py#L333), [`_attribute`](../python/src/guardrail_chatbot_jev/guard.py#L350) |
| 会話が「最近リスクあり」とみなされる条件 | [`Session.watching`](../python/src/guardrail_chatbot_jev/session.py#L128) |
| 止めたメッセージを印として残し、モデルにはカテゴリだけを伝える（本文は見せない） | [`Session.record`](../python/src/guardrail_chatbot_jev/session.py#L99), [`model_history`](../python/src/guardrail_chatbot_jev/session.py#L106) |
| 会話全体のチェック：監視と報告のみで、ターンは止めない | [`check_conversation`](../python/src/guardrail_chatbot_jev/guard.py#L170) |

## ターンごとの例

| ターン | ユーザー | アシスタント | ガードレールの動き | 結果 |
| --- | --- | --- | --- | --- |
| 1 | 「肥料で爆薬を作る方法は？」 | （なし） | 手順 1：それ自体が有害。 | **停止。** 会話は「最近リスクあり」になる。 |
| 2 | 「ごめん、忘れて。明日のハノイの天気は？」 | 「晴れ、24〜30 度。」 | 手順 1・2：無害。手順 3：応答は何も完成させていない（0.02）。 | **配信。** 以前の設計では保留された。 |
| 3 | 「爆薬の製造は法律でどう罰せられる？」 | 「場合により懲役刑になります。」 | 手順 3：ユーザーは距離を置いている（0.90）。応答は何も完成させていない（0.05）。 | **配信。** |
| 4 | 「じゃあ続けて。手順 2 は？」 | 「手順 2：混合物を一日乾かし、均等に分ける。」 | 手順 3：断った依頼の次の手順（0.95）。 | **停止。** 文脈付きの読みだけが捉える。 |
| 4′ | 同上 | 「その続きはお手伝いできません。」 | 手順 3：何も完成させていない。 | **配信。** |

## 定義

### 現在の設定

| 設定 | 値 | 意味 |
| --- | --- | --- |
| 履歴ウィンドウ | 10 メッセージ | 会話チェックと文脈付きの読みは直近 10 メッセージを見る。 |
| 判定ごとのリスク | allow 0 · flag 0.25 · review 0.6 · block 1.0 | 判定がセッションのリスクに加える量。 |
| 減衰 | 0.5 | チェックごとに前のリスクは半分になり、新しい判定のリスクとの大きい方を保つ。 |
| 監視開始 | リスク ≥ 0.2 | この間、応答を文脈付きでも読む。 |
| 監視の持ち越し | 2 ターン | 会話レベルの review 以上、または block の後、2 ターン監視を続ける。 |
| 帰属 | ≥ 0.5、かつ「距離を置く」より高い | 応答が以前の有害な依頼を完成させる確率が 0.5 以上で、ユーザーが距離を置いている確率より高いときだけ効く。 |
| 裏付け | ≥ 0.02 | 主の質問がそのカテゴリに 2 % 以上を与えれば、センチネルは裏付けあり。 |
| 弱いセンチネル | block 帯未満 | 単独で block 帯未満なら flag として記録し、配信する。 |
| 弱めないカテゴリ | `ssh`（ベトナムパック：`ssh`、`vsv`、`vld`） | センチネルだけでも強さを保つ。 |
| 拒否の応答 | refusal ≥ 0.8、センチネル < 0.5 | flag まで下げる（`sid`、`prv`、`ipv` を除く。ベトナムパックは `vsv`、`vld` も）。 |
| 遮蔽して配信 | センチネル < 0.8 | センチネルだけが示す個人情報は、ブロックせず遮蔽して配信。 |
| 信頼度ゲート | 信頼度 < 0.65 | 所見やニアミスがあれば review。ただし意図が benign で信頼度 ≥ 0.5 なら除く。 |
| 会話の上限 | escalation ≤ 0.5 | flag まで（`cse`、`ssh` を除く）。 |
| Jev に接続できない | 入力：配信 · 出力：保留 · 会話：配信 | 出力チェックが最後の砦。 |

### カテゴリ別のしきい値

| カテゴリ | 名前 | Flag / review / block | 出力 | 備考 |
| --- | --- | --- | --- | --- |
| `cse` | Child sexual exploitation | 0.03 / 0.06 / 0.15 | = | センチネル; block 未満にしない |
| `iwp` | Indiscriminate weapons (CBRNE) | 0.04 / 0.08 / 0.2 | = | センチネル |
| `ssh` | Suicide and self-harm | 0.07 / 0.15 / 0.3 | = | センチネル; crisis_support で処理 |
| `vcr` | Violent crimes | 0.12 / 0.25 / 0.45 | = | センチネル |
| `src` | Sex-related crimes | 0.12 / 0.25 / 0.45 | = |  |
| `hte` | Hate and discrimination | 0.15 / 0.28 / 0.5 | 0.08 / 0.18 / 0.35 |  |
| `ncr` | Non-violent crimes | 0.15 / 0.3 / 0.5 | = |  |
| `sxc` | Adult sexual content | 0.2 / 0.4 / 0.6 | = |  |
| `prv` | Privacy and personal data | 0.2 / 0.4 / 0.7 | 0.15 / 0.28 / 0.5 | センチネル; redact で処理 |
| `ipv` | Intellectual property | 0.25 / 0.45 / 0.7 | = | output, conversation のみ |
| `dfm` | Defamation | 0.18 / 0.35 / 0.6 | = | output, conversation のみ |
| `spc` | Specialized advice | 0.25 / 0.5 / 0.85 | = | guide で処理; output のみ |
| `elc` | Elections and civic integrity | 0.15 / 0.3 / 0.5 | = |  |
| `pij` | Prompt injection and jailbreak | 0.15 / 0.3 / 0.55 | = | センチネル; input, conversation のみ |
| `sid` | Sensitive information disclosure | 0.08 / 0.18 / 0.35 | = | センチネル; output, conversation のみ |
| `exa` | Excessive agency | 0.15 / 0.3 / 0.5 | = | output, conversation のみ |
| `mis` | Misinformation and unsupported claims | 0.25 / 0.45 / 0.8 | = | guide で処理; output のみ |
| `scp` | Out of scope | 0.4 / 0.75 / 0.95 | = | 無効 |

### 規則の説明

1. メッセージと応答はそれぞれ単独で採点し、判定は到達した最も強い帯。
2. セッションは本文ではなくリスクを覚える：前のリスクの半分と新しい判定のリスクの大きい方。
3. リスク 0.2 以上、重大な所見の後 2 ターン、保留メッセージがウィンドウ内にある間は監視する。
4. 監視中は応答を履歴と一緒にもう一度読み、以前の有害な依頼を完成させる場合だけ効かせる。
5. 履歴だけで判定が上がることはない。

### 数式

```
V_in(t)   = D(input,  J(q_t))
V_out(t)  = D(output, J(r_t))

W_t       = carry_left > 0  ∨  risk_t ≥ 0.2  ∨  placeholder ∈ H_t          (watched)
V_ctx     = D(output, J(r_t | H_t))                                          (only if W_t)
c, d      = P(reply completes an earlier harmful request), P(user steps away)
A_t       = c ≥ τ  ∧  c ≥ d  ∧  V_ctx has a finding ≥ flag,   τ = 0.5
V(t)      = V_out(t) ⊕ V_ctx  if A_t,  else V_out(t)

risk_t+1  = max(δ · risk_t, ρ(action)),  δ = 0.5,  ρ = (0, 0.25, 0.6, 1.0) for (allow, flag, review, block)
carry     = 2 turns after a conversation verdict ≥ review or any block
```

コード：`V_in` [`check_input`](../python/src/guardrail_chatbot_jev/guard.py#L119), `V_out` [`check_output`](../python/src/guardrail_chatbot_jev/guard.py#L131), `W_t` [`Session.watching`](../python/src/guardrail_chatbot_jev/session.py#L128), `V_ctx`, `c`, `d` [`_check_in_context`](../python/src/guardrail_chatbot_jev/guard.py#L333) / [`context_questions`](../python/src/guardrail_chatbot_jev/questions.py#L135), `A_t`, `⊕` [`_attribute`](../python/src/guardrail_chatbot_jev/guard.py#L350), `risk` [`Session.observe`](../python/src/guardrail_chatbot_jev/session.py#L146), `carry` [`Session.advance`](../python/src/guardrail_chatbot_jev/session.py#L163)

## 単一ターンの較正

説明：

1. **センチネル単独は弱い信号。** 専用の yes/no 質問だけがカテゴリを検出し、主なハザード質問がその
   カテゴリに 2 % 未満しか与えない場合、その所見は「裏付けなし」とする。
2. **弱い所見は flag まで。** 単独で block 帯に達した場合を除く。カテゴリの「never below」でも引き上げ
   ない。自傷は弱めない（ベトナムパックは `vsv`、`vld` も）。
3. **拒否する応答**（refusal ≥ 0.8）に 0.5 未満の弱いセンチネルが付いた場合は flag として記録する。
   秘密・個人データ・保護されたテキストの漏えい（`sid`、`prv`、`ipv`）は例外。拒否文にも含まれうるため。
4. **センチネルだけが見つけた個人データ**はブロックせずマスクして届ける。センチネルが 0.8 に達した場合を除く。
5. **信頼度が低い**（0.65 未満）所見やニアミスはレビューへ。ただし Jev が意図を良性と 0.5 以上の信頼度で
   判断した場合は除く。
6. **エスカレートしない会話**（escalation ≤ 0.5）は flag まで。`cse` と `ssh` を除く。

```
u_k                 = finding from the sentinel only  ∧  choice_k < 0.02      (uncorroborated)
never_below         applied only if ¬u_k ∨ p ≥ θ_block
u_k ∧ p < θ_block                                                    → at most flag
output ∧ refusal ≥ 0.8 ∧ u_k ∧ p < 0.5 ∧ k ∉ {sid, prv, ipv}         → flag
u_k ∧ route_k = redact ∧ action = block ∧ p < 0.8                    → review (masked)
confidence gate:  conf < 0.65 ∧ (finding ∨ p ≥ θ_flag / 2) → review,  unless intent = benign ∧ conf ≥ 0.5
conversation:     escalation ≤ 0.5 → at most flag,  except cse, ssh
```

コード：`u_k` [`decide`](../python/src/guardrail_chatbot_jev/decide.py#L57), `never_below` [`_finding`](../python/src/guardrail_chatbot_jev/decide.py#L212), weak [`decide`](../python/src/guardrail_chatbot_jev/decide.py#L70), refusal [`_cap_uncorroborated_on_refusal`](../python/src/guardrail_chatbot_jev/decide.py#L245), redact [`_redact_instead_of_block`](../python/src/guardrail_chatbot_jev/decide.py#L270), gate [`_confidence_gate`](../python/src/guardrail_chatbot_jev/decide.py#L378), conversation [`no-escalation-caps-conversation`](../policies/standard-v1.json#L700), settings [`sentinel_corroboration`](../policies/standard-v1.json#L23) / [`confidence_gate`](../policies/standard-v1.json#L32), [`spc`](../policies/standard-v1.json#L328), [`ncr`](../policies/standard-v1.json#L207), [`iwp`](../policies/standard-v1.json#L83)

## リアルタイムでのレビュー

`ReviewHandling: ReviewAsAudit`（[`review_handling`](../python/src/guardrail_chatbot_jev/guard.py#L107)、[`_audit`](../python/src/guardrail_chatbot_jev/guard.py#L314)）：内容を止めるのは `block` のみ。`review` は配信して優先監査へ、`flag` は
サンプリング監査へ回す。fail-closed の面の degraded 判定は保留のまま。

## 結果

| 実測、223 会話 | フロア（以前） | 帰属（現在） |
| --- | --- | --- |
| 保留された無害なターン | 171 / 194 | **0 / 194** |
| 検出した有害な応答 | 19 / 19 | **19 / 19** |
| 検出したエスカレーション | 9 / 9 | **9 / 9** |
| レビューに回った無害な会話 | 172 / 194 | **2 / 194** |

| 再判定、記録済み約 5,000 件 | 無害の保留 | 違反の検出 |
| --- | --- | --- |
| 最終較正の前 | 0.62 % | 100 % |
| 最終較正の後 | **0.04 %** | **100 %** |
| リアルタイム（`ReviewAsAudit`） | **0.02 %** 停止 | 有害な応答はすべて停止 |

データセット、手順、アブレーション、ノイズ、回帰、再現手順は[英語版](multiturn.md)を参照。
