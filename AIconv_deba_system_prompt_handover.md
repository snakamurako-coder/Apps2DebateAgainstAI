# システムプロンプト引き継ぎ書
## AI英会話 & パーラメンタリーディベート

> 作成日: 2026-09-01  
> 対象プロジェクト: `App2DebateAgainstAI`（Google Apps Script + HTML Web App）

---

## 凡例

| 記号 | 意味 |
|------|------|
| 🤖 AI のみ | システムプロンプト / messages に含まれ、ユーザーには表示されない |
| 👤 ユーザーのみ | UI に表示されるが、AI への messages には含まれない |
| 🔁 両方 | AI への messages にも含まれ、かつ UI にも表示される |
| 🔧 内部のみ | AI にも UI にも渡されない（ファイル名・管理・計算用途） |

---

## 概要

このプロジェクトは Google Apps Script 上で動作する Web アプリ。  
スプレッドシートに記述された設定値を読み込み、OpenAI API（`gpt-4o-mini`）にシステムプロンプトを構築して渡す。  
**「AI英会話」と「パーラメンタリーディベート」の2モードがある。**

---

## 共通の基盤

### API呼び出し関数 `chat_(messages, opts)`
[コード.js L1055–L1086](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1055-L1086)

```
messages = [
  { role: 'system', content: <システムプロンプト> },
  { role: 'user',   content: <ユーザー発話> },
  { role: 'assistant', content: <AI応答> },
  ...
]
```

- モデル: `gpt-4o-mini`（定数 `OPENAI_MODEL`）
- temperature: `0.7`（定数 `OPENAI_TEMPERATURE`）
- APIキー: スクリプトプロパティ `OPENAI_API_KEY` に保存

---

## ① AI英会話モード

### スプレッドシートの構造と「誰に見えるか」

**フォルダ**: `Themes/`（スクリプトの親フォルダ直下）  
**シート名**: `Assignments`  
**ヘッダー定義** [コード.js L10–L14](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L10-L14):

| 列名 | 変数名（JS） | 誰に見えるか | 用途・詳細 |
|------|-------------|:-----------:|-----------|
| 通し番号 | `serial` | 🔧 内部のみ | ファイル名生成に使用 |
| 見出し | `title` | 👤 ユーザーのみ | 課題選択ドロップダウンに表示 |
| **生成AIへの指示文** | `sys` | 🤖 AI のみ | `Instructions: {値}` としてシステムプロンプトに挿入 |
| **英会話の状況設定** | `context` | 🤖 AI のみ | `Context: {値}` としてシステムプロンプトに挿入 |
| 生徒への表示文 | `displayText` | 👤 ユーザーのみ | UIの「生徒への指示」テキストエリアに表示。AIには渡さない |
| 役割A | `roleA` | 🤖 AI のみ | システムプロンプト内のロール名として使用 |
| **Aの最初の台詞** | `roleAFirst` | 🔁 両方 | AIの第1発話を完全一致で強制。UIには「最初のセリフ参考」として表示 |
| 役割B | `roleB` | 🤖 AI のみ | システムプロンプト内のロール名として使用 |
| **Bの最初の台詞** | `roleBFirst` | 🔁 両方 | AIの第1発話を完全一致で強制。UIには「最初のセリフ参考」として表示 |
| **フィードバックの仕方** | `feedbackStyle` | 🤖 AI のみ | フィードバック生成AIの `Preferences: {値}` に挿入。UIには表示しない |
| フィードバック直前の提示情報 | `preFeedbackTip` | 👤 ユーザーのみ | UIのフィードバック前ヒントとして表示。AIには渡さない |
| 問題文 | `prompt` | 👤 ユーザーのみ | UIの「問題文」に表示（displayText より優先）。AIには渡さない |
| 備考 | `note` | 🔧 内部のみ | 管理者用メモ。誰にも渡さない |

> [!IMPORTANT]
> `displayText.value = currentRow.prompt || currentRow.displayText || ''`  
> UIには `prompt`（問題文）が優先表示され、なければ `displayText`（生徒への表示文）が使われる。  
> **どちらも AI へは渡らない。**

### データの流れ

```
スプレッドシート Themes/Assignments シート
        ↓  listThemeProblems() で行データを読み込む
     row オブジェクト (JS)
        ↓  startConversation() / continueConversation()
   buildSystemPrompt_({ cefr, userRoleName, row })
        ↓  messages 配列に system メッセージとして追加
        ↓  chat_() で OpenAI API へ送信
       AI の返答
```

### システムプロンプト生成 `buildSystemPrompt_()`
[コード.js L1146–L1167](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1146-L1167)

生成されるプロンプトの構造（各行を `\n` で連結）:

```
You are an English conversation partner for CEFR {cefr} learners.
Roles: Learner = {studentRoleName}; Assistant = {aiRoleName}.
Context: {row.context}                     ← 🤖「英会話の状況設定」
Instructions: {row.sys}                    ← 🤖「生成AIへの指示文」
Strict Rules:
 - Always stay strictly in your assigned role: you are the {aiRoleName}. ...
 - Never speak lines that belong to the {studentRoleName} role ...
 - Keep turns short and simple (CEFR {cefr}).
 - Ask one question at a time ONLY if it is natural for the {aiRoleName} ...
 - If the learner is confused, rephrase more simply.
 - Only when it appears for the first time ...(外来語ルール)
 - Do not add extra sentences that switch roles or move the scene ahead on your own.
Begin immediately. On your first turn ONLY, reply with EXACTLY: "{aiFirstLine}"
                                            ← 🔁 roleAFirst または roleBFirst
```

> [!NOTE]
> `row.context`（状況設定）と `row.sys`（指示文）が空の場合は、その行自体がプロンプトに追加されない（空スキップ）。

### 役割決定ロジック `getEffectiveRolePack_()`
[コード.js L1115–L1143](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1115-L1143)

- `roleB` / `roleBFirst` が空の場合 → **ロール固定モード**  
  AI = 役割A（roleA）、生徒 = 役割B（roleB）  
- どちらも入っている場合 → フロントが指定した `userRoleName` に従い AI/人間を割り当て

### 会話開始 `startConversation()`
[コード.js L1345–L1350](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1345-L1350)

1. `buildSystemPrompt_()` でシステムプロンプト生成
2. `chat_()` に `[{ role:'system', content: sys }]` のみ送信
3. AIが最初の一言を返す（roleAFirst / roleBFirst を完全一致で強制）

### 会話継続 `continueConversation()`
[コード.js L1357–L1371](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1357-L1371)

1. 毎回 `buildSystemPrompt_()` を再生成
2. `system` + フロントから届いた `user/assistant` 履歴を連結して送信
3. `keepPairs: 4`（直近4往復のみ保持）

### フィードバック生成 `requestFeedback()`
[コード.js L1203–L1233](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1203-L1233)

```
system: "You are an English teacher.
You will receive a full dialogue transcript ...
Give concise, constructive feedback for a CEFR {cefr} learner.
Preferences: {row.feedbackStyle}           ← 🤖「フィードバックの仕方」

STRICT RULES:
- Evaluate ONLY lines starting with 'User({userRoleName}):'
- DO NOT evaluate 'AI({aiRoleName}):' lines
- Use Japanese.
- Keep it short and actionable (2–5 bullets).
- Start your reply with 'フィードバック：'"

user: <会話ログ（User/AI ラベル付き）>
```

---

## ② パーラメンタリーディベートモード

### スプレッドシートの構造と「誰に見えるか」

**フォルダ**: `Debates/`（スクリプトの親フォルダ直下）  
**シート名**: `Assignments`  
**ヘッダー定義** [コード.js L26–L32](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L26-L32):

| 列名 | 変数名（JS） | 誰に見えるか | 用途・詳細 |
|------|-------------|:-----------:|-----------|
| 通し番号 | `serial` | 🔧 内部のみ | ファイル名生成に使用 |
| 見出し | `title` | 👤 ユーザーのみ | 課題選択ドロップダウンに表示 |
| **ディベートテーマ** | `motion` | 🔁 両方 | `Motion: {値}` としてAIに渡す。UIにも題目として表示 |
| 生徒への表示文 | `displayText` | 👤 ユーザーのみ | UIの説明欄に表示。AIには渡さない |
| **生成AIへの指示文** | `sys` | 🤖 AI のみ | `General instructions: {値}` としてスピーチプロンプトに挿入。空ならデフォルト定数 |
| PM秒〜GR秒 | `pmSec`〜`grSec` | 🔁 両方 | タイマー表示（UI）＋ AIのターゲット語数計算（`秒/60*150`語）に使用 |
| 作戦タイム1・2秒 | `prep1Sec`, `prep2Sec` | 👤 ユーザーのみ | UIのタイマーのみ。AIへは渡さない |
| **AI反論強度** | `aiRebuttalStrength` | 🤖 AI のみ | 1〜5を自然言語に変換し `Rebuttal intensity: ...` としてプロンプトに挿入 |
| **AI論理厳密度** | `aiLogicTightness` | 🤖 AI のみ | 1〜5を自然言語に変換し `Logical rigor: ...` としてプロンプトに挿入 |
| AI TTS速度 | `aiTtsRate` | 👤 ユーザーのみ | フロントの音声読み上げ速度として使用。AIへは渡さない |
| **ジャッジ用プロンプト** | `judgePrompt` | 🤖 AI のみ | ジャッジAIの `Additional criteria: {値}` として挿入。スピーチAIには渡さない |
| **PM〜GR用プロンプト** | `speechPrompts.PM`〜`GR` | 🤖 AI のみ | 各スピーチ固有の `Speech-specific instructions: {値}` として挿入 |
| 備考 | `note` | 🔧 内部のみ | 管理者用メモ。誰にも渡さない |

### データの流れ

```
スプレッドシート Debates/Assignments シート
        ↓  listDebateSets() で行データを読み込む
     row オブジェクト (JS)
        ↓  generateDebateSpeech_() が呼ばれる（AIスピーチ担当ごとに1回）
   buildDebateSpeechPrompt_(row, speechId, side, cefr)
        ↓  messages 配列に構築して chat_() へ送信
       AI のスピーチ文
```

### AIスピーチ生成プロンプト `buildDebateSpeechPrompt_()`
[コード.js L1740–L1772](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1740-L1772)

生成されるシステムプロンプトの構造:

```
You are competing in a parliamentary debate.
Motion: {row.motion}                               ← 🔁「ディベートテーマ」
You are delivering the {step.label} speech for the {sideLabel} side.
General instructions: {row.sys}                    ← 🤖「生成AIへの指示文」
Speech-specific instructions: {speechPrompts[id]}  ← 🤖「PM〜GR用プロンプト」
Rebuttal intensity: <自然言語>                      ← 🤖「AI反論強度」1〜5変換
Logical rigor: <自然言語>                           ← 🤖「AI論理厳密度」1〜5変換
Language level: {buildDebateCefrHint_(cefr)}        ← CEFRレベルに応じた英語制限
Speech type rules: {typeRules[step.speechType]}    ← OREO/Rebuttal/Reply の型
You have NO preparation time. ...
Deliver ONLY the speech text in English. ...
Target length: approximately {word数} words.       ← 🔁「PM秒」などから計算
```

### generateDebateSpeech_() の messages 構造
[コード.js L1809–L1839](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1809-L1839)

```js
[
  { role: 'system', content: <buildDebateSpeechPrompt_() の出力> },
  { role: 'user',   content: 'Previous speeches in this debate:\n\n' + history },
                             // ← ラベル付き全スピーチ履歴（最大12000文字）
  { role: 'user',   content: `Now deliver your ${speechId} speech ...` }
]
```

- `keepAll: true` で全履歴を保持（ペア数制限なし）
- `maxTokens`: 持ち時間(秒)から計算（350〜800トークン）

### 強度パラメータの変換 `aiStrengthToText_()`
[コード.js L1703–L1724](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1703-L1724)

| 値 | Rebuttal（反論強度） | Logic（論理厳密度） |
|----|--------------------|--------------------|
| 1 | Use minimal rebuttal; focus on your own case. | You may leave minor logical gaps; prioritize persuasion. |
| 2 | Rebut only the strongest opposing point briefly. | Mostly coherent; occasional leaps are acceptable. |
| 3 | Rebut key opposing arguments clearly. | Maintain solid logical structure throughout. |
| 4 | Rebut thoroughly and challenge weak logic in opposing speeches. | Tight logic with no unsupported claims. |
| 5 | Aggressively rebut every major opposing claim with evidence. | Maximum rigor: every claim must be supported; expose logical flaws. |

### ジャッジ生成 `judgeDebate_()`
[コード.js L1842–L1868](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L1842-L1868)

```
system: "You are an impartial parliamentary debate judge.
Judge ONLY based on arguments in the transcript.
Do NOT mention human/AI.
Output JSON ONLY: {"winnerSide":"gov"|"opp"|"draw","verdictJa":"..."}
verdictJa: concise Japanese explanation (3-6 sentences).
{row.judgePrompt}                                  ← 🤖「ジャッジ用プロンプト」"

user: <匿名化されたトランスクリプト（Motion + ラベル付き全スピーチ）>
```

---

## デフォルト指示文

`sys`（生成AIへの指示文）が空の場合に使われるフォールバック定数:

[コード.js L33–L38](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js#L33-L38)

```
Constructive speeches (PM, LO): Use OREO (Opinion, Reason, Example, restated Opinion).
Include exactly 2 reasons; give one brief example for each reason.
Rebuttal speeches (MG, MO): Rebut at least 2 points from the other side, then add 1 new supporting point for your case.
Reply speeches (OR, GR): No new arguments. Summarize the clash and explain why your side wins.
AI debaters have no preparation time: use all prior speeches and respond immediately.
```

---

## Claude でシステムプロンプトを作成する際のポイント

### AI英会話用

| フィールド | 誰に見えるか | 挿入箇所 | 注意点 |
|-----------|:-----------:|---------|--------|
| `英会話の状況設定` | 🤖 AI のみ | `Context: {値}` | 場所・状況の設定。短く明確に |
| `生成AIへの指示文` | 🤖 AI のみ | `Instructions: {値}` | Strict Rules の後ろに追加。矛盾しないよう注意 |
| `役割A` / `役割B` | 🤖 AI のみ | `Roles: Learner = ...; Assistant = ...` | AI が演じる役割名 |
| `Aの最初の台詞` / `Bの最初の台詞` | 🔁 両方 | `Begin immediately. On your first turn ONLY, reply with EXACTLY: "..."` | 完全一致で強制される |
| `フィードバックの仕方` | 🤖 AI のみ | `Preferences: {値}` | フィードバックAIへの指示。例: "Focus on grammar." |
| `生徒への表示文` | 👤 ユーザーのみ | UIの説明欄 | AIには届かない。学習者への課題説明に使う |
| `問題文` | 👤 ユーザーのみ | UIの問題欄（displayText より優先） | AIには届かない |

> [!TIP]
> `生成AIへの指示文`（AI英会話）は Strict Rules の**後ろに** `Instructions:` として追加されるため、Strict Rules と矛盾する内容を書くと Strict Rules が優先される傾向がある。Strict Rules を上書きしたい場合はコード側の修正が必要。

### ディベート用

| フィールド | 誰に見えるか | 挿入箇所 | 注意点 |
|-----------|:-----------:|---------|--------|
| `ディベートテーマ` | 🔁 両方 | `Motion: {値}` | English で記述推奨。UIにも表示 |
| `生成AIへの指示文` | 🤖 AI のみ | `General instructions: {値}` | 全スピーチ共通。空ならデフォルト定数 |
| `PM〜GR用プロンプト` | 🤖 AI のみ | `Speech-specific instructions: {値}` | スピーチタイプ固有の追加指示 |
| `AI反論強度` | 🤖 AI のみ | `Rebuttal intensity: ...` | 1〜5の数値。3が標準 |
| `AI論理厳密度` | 🤖 AI のみ | `Logical rigor: ...` | 1〜5の数値。3が標準 |
| `ジャッジ用プロンプト` | 🤖 AI のみ | ジャッジAIの `Additional criteria: {値}` | ジャッジ専用。スピーチAIには渡らない |
| `AI TTS速度` | 👤 ユーザーのみ | 音声読み上げ速度 | AIには届かない。0.6〜2.0 |
| `生徒への表示文` | 👤 ユーザーのみ | UIの説明欄 | AIには届かない |

---

## ファイル構成早見表

| ファイル | 役割 |
|---------|------|
| [コード.js](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/コード.js) | GAS バックエンド全体 |
| [index.html](file:///c:/Users/enhan/コード開発/Projects/App2DebateAgainstAI/index.html) | フロントエンド（HTML+JS） |
| `Themes/` フォルダ | AI英会話用スプレッドシートを格納 |
| `Debates/` フォルダ | ディベート用スプレッドシートを格納 |
| `inbox_submissions/` フォルダ | 提出データ（JSON）を格納 |

## 重要な関数一覧

| 関数名 | 行 | 説明 |
|--------|----|------|
| `buildSystemPrompt_()` | L1146 | AI英会話のシステムプロンプト生成 |
| `buildDebateSpeechPrompt_()` | L1740 | ディベートスピーチのシステムプロンプト生成 |
| `requestFeedback()` | L1203 | フィードバック用プロンプト構築・API呼び出し |
| `judgeDebate_()` | L1842 | ジャッジ用プロンプト構築・API呼び出し |
| `chat_()` | L1055 | OpenAI API 呼び出し共通関数 |
| `listThemeProblems()` | L989 | スプレッドシートから英会話課題を読み込む |
| `listDebateSets()` | L1627 | スプレッドシートからディベート設定を読み込む |
