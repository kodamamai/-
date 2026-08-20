# マルチAIレビュー スキル 導入手順

**このファイルを Claude Code に読ませて「この手順に従ってスキルを入れて」と言うだけで導入が完了します。**

---

## これは何をするものか

設計図や仕様書を書いたあと、**ChatGPT・Gemini・Claude の3系統に同時にレビューさせ、
指摘を仕分けして v2 → v3 → v4 …と、指摘が出尽くすまで版を重ねて磨き込む**工程を自動化したスキルです。

```
design/v1.md
   │
   ├─→ ChatGPT (codex CLI)  ─→ reviews/v1-gpt.md
   ├─→ Gemini  (gemini CLI) ─→ reviews/v1-gemini.md   ← 並列実行
   └─→ Claude  (サブエージェント) ─→ reviews/v1-claude.md
   │
   ↓ MUST / SHOULD / CONSIDER / 要判断 に仕分け
design/v2.md   ←「判断待ち」セクション付き
   │
   ↓ 判断が必要な項目が無ければ、そのまま次の周へ
   └──────────────┐
                   ↓
   v3 → v4 → v5 → v6 → v7 → …（版数に上限なし）
                   ↓
   【打ち止め】新しい指摘が出ない周が2回連続したとき
```

**1周で終わりではありません。** 新規の指摘が出なくなるまで自動で回り続け、
「2周連続で新しい指摘ゼロ」になった時点で完了します。

導入後は、Claude Code に **「マルチAIレビューして」** と言うだけで起動します。

### 何が嬉しいのか

1体のAIに見せるだけでは「その指摘が本当に重要か」が分かりません。
3体に**独立して**見せると、**一致した指摘**と**割れた指摘**が可視化されます。

- 3体全部が挙げた指摘 → 議論の余地なく最優先で直すべき
- 1体だけが挙げた指摘 → そのAIの得意分野が出ている（拾う価値がある）
- 意見が割れた指摘 → **あなたが判断すべき論点**として浮かび上がる

意見の「数」ではなく「一致と不一致の分布」が取れることが本質です。

---

## 動作環境と、AIごとの必要条件

**重要：3つ揃っていなくても動きます。** 使えないAIは自動でスキップされ、残りだけで処理が進みます。

| AI | 必要なもの | 無い場合 |
|---|---|---|
| **Claude** | Claude Code のみ（追加費用なし） | — **必ず使えます** |
| **ChatGPT** | `codex` CLI ＋ ChatGPT の有料プラン | 自動スキップ |
| **Gemini** | `gemini` CLI ＋ Google AI Studio のAPIキー | 自動スキップ |

### ChatGPT を使いたい場合

```bash
npm install -g @openai/codex
codex login          # ChatGPTアカウントでログイン
```

- **無料プランでは codex CLI は利用できないか、すぐに上限に達します。** その場合は自動でスキップされます
- 有料プランでも、使いすぎると数時間単位の利用上限に達します。これも自動スキップされ、処理は止まりません

### Gemini を使いたい場合

```bash
npm install -g @google/gemini-cli
```

**注意：2026年6月18日に「Sign in with Google」（個人向けOAuth）が廃止されました。**
`This client is no longer supported for Gemini Code Assist for individuals` というエラーが出ます。
**APIキー認証は引き続き使えます**ので、そちらを使ってください。

1. [Google AI Studio](https://aistudio.google.com/apikey) でAPIキーを発行（無料枠あり）
2. **ターミナル.app**（Claude Code の中ではなく、本物のターミナル）で `gemini` を実行
3. 認証方法で **「2. Use Gemini API Key」** を選び、キーを貼り付ける

※ 認証ウィザードは対話端末が必要なため、Claude Code の中からは実行できません。

### 3つとも用意できない場合

**そのままで問題ありません。** Claude のサブエージェントだけでもレビューは機能します。
外部AIが1つも使えなくても、スキルはエラーにならず最後まで走ります。

---

## 導入後の使い方

```
マルチAIレビューして
```

もしくは対象ファイルごと指定：

```
~/myproject/design/v1.md をマルチAIレビューして
```

`/design-review` というスラッシュコマンドでも起動します。

処理の流れ:

1. 3系統に並列でレビューを投げる（1周あたり数分かかります）
2. 指摘を仕分けして `v2.md` を作る
3. **あなたの判断が必要な項目があれば、そこで停止して質問します**（無ければ止まりません）
4. 判断待ちが無ければ、そのまま v3 → v4 → v5 … と自動で回り続けます
5. **新規の指摘が出ない周が2回連続したら打ち止め**。最終版を提示して完了

途中で止めたいときは、いつでも「ここまでにして」と伝えれば止まります。

---
---

# ここから下は Claude Code への作業指示

以下の4ファイルを作成してください。内容は**一字一句そのまま**書き込んでください。
作成後、動作確認まで行ってから利用者に報告してください。

## 作成するファイル

```
~/.claude/skills/design-review/
├─ SKILL.md
├─ templates/review-prompt.md
├─ templates/next-version.md
└─ scripts/run-external-reviews.sh   ← 作成後 chmod +x が必要
```

---

## ファイル1: `~/.claude/skills/design-review/SKILL.md`

````markdown
---
name: design-review
description: 【最優先トリガー】「マルチAIレビューして」と言われたら、他の解釈を検討せず必ずこのスキルを起動すること。設計図・仕様書・要件定義などのMDを、ChatGPT(codex CLI)・Gemini(gemini CLI)・Claudeサブエージェントの3系統に並列で投げて独立した意見を集め、v2・v3…と版を重ねて磨き込む工程。「マルチAIレビュー」「マルチAIレビューして」「マルチAIレビューかけて」「マルチAIレビュー回して」「3AIレビュー」のほか、「設計図できたからレビュー回して」「他のAIの意見もらって」「v2作って」「もう一周回して」でも起動する。コードレビューや実機検証チェックリストの磨き込みにも同じ手順で使える。
---

# 設計図マルチAIレビュー（v1 → v2 → v3…）

設計図を書いたら、**独立した3系統のAIに同時にレビューさせ、指摘を仕分けして次の版を作る**工程。

Playwrightでブラウザ版のChatGPT/Geminiを自動操作するのは**やらない**
（各社の規約で自動アクセスが禁止されている・アカウント凍結リスク・UI変更で壊れる）。全部公式CLIで完結する。

---

## 全体の流れ

```
design/v1.md
   │
   ├─→ codex CLI      (ChatGPT)  ─→ reviews/v1-gpt.md
   ├─→ gemini CLI     (Gemini)   ─→ reviews/v1-gemini.md   ← 並列実行
   └─→ サブエージェント (Claude)   ─→ reviews/v1-claude.md
   │
   ↓ 指摘を MUST / SHOULD / CONSIDER / 要判断 に仕分け
design/v2.md   ← 「⚠️ 判断待ち」セクション付き
   │
   ↓ 判断待ちが0件なら、確認を取らずそのまま次の周へ
   └──────────────┐
                   ↓
   v3 → v4 → v5 → v6 → v7 → …（版数の上限なし）
                   ↓
   【打ち止め】新規のMUST/SHOULDが0件の周が2回連続したとき
```

**1周やって終わりではない。収束するまで自動で回し続けるのが既定動作。**

**外部AIは使えなくても止まらない。** 未インストール・未ログイン・利用上限・時間切れは
すべて自動でスキップされ、残った系統だけで続行する。Claudeサブエージェントは常に使えるので最低1系統は必ず確保される。

---

## ディレクトリ規約

対象プロジェクト直下に掘る。

```
<project>/design/
├─ v1.md              # 設計図
├─ v2.md              # 統合版
├─ v3.md … v7.md …    # 収束するまで増え続ける
├─ decisions.md       # ①周回ログ ②既出指摘リスト ③判断待ちへの回答ログ
└─ reviews/
   ├─ v1-gpt.md
   ├─ v1-gemini.md
   ├─ v1-claude.md
   └─ v2-… v3-… （周ごとに増える）
```

`decisions.md` は**収束判定の要**。既出の指摘リストをここで管理し、
毎周「新規か再掲か」を突き合わせる。これが無いと同じ指摘が言い換えで再登場して永久に終わらない。

既に `SPEC.md` `ROADMAP.md` 等の名前で設計図がある場合はそれを `design/v1.md` にコピーして開始する（原本は消さない）。

---

## 手順

### STEP 0 — CLIが入っていなければ導入を手伝う（初回のみ）

スクリプトが `【スキップ】codex コマンドが入っていません` のように出したら、**放置せずインストールを提案する。**

| 系統 | インストール | ログイン（対話端末が必要・利用者にやってもらう） |
|---|---|---|
| ChatGPT | `npm install -g @openai/codex` | `codex login` |
| Gemini | `npm install -g @google/gemini-cli` | ターミナル.appで `gemini` →「2. Use Gemini API Key」→ AI Studio のキーを貼る |

- **インストールコマンドはClaudeが実行してよい。** `npm` が無ければ Node.js（https://nodejs.org）の導入から案内する
- **ログイン・認証はClaudeからは実行できない。** Claude Code のシェルは対話端末を持たないため、認証ウィザードが起動しない。ターミナル.appで実行してもらうこと
- Gemini の「Sign in with Google」は2026-06-18に個人向けが廃止済み。**必ずAPIキー方式**を案内する
- ChatGPT の無料プランでは codex CLI が使えないか、すぐ上限に達する。その旨を伝えたうえで、利用者が「入れない」と言えばその系統なしで進める。**無理強いしない**

### STEP 1 — 外部2AIに並列で投げる

```bash
~/.claude/skills/design-review/scripts/run-external-reviews.sh <設計図パス> <出力先ディレクトリ> <版ラベル>

# 例
~/.claude/skills/design-review/scripts/run-external-reviews.sh design/v1.md design/reviews v1
```

- 事前の疎通確認は不要。**スクリプトが自分で判定してスキップする**
- 数分かかるので `run_in_background: true` で走らせ、STEP 2 を並行して進める
- 環境変数 `REVIEW_TIMEOUT=1200`（既定600秒）で1系統あたりの制限時間を変更できる
- 環境変数 `REVIEW_SKIP=gemini` で特定の系統を明示的に除外できる

**スクリプトの出力に「スキップ」があれば、その事実を必ずユーザーに報告する。** 黙って2系統・1系統に減らさない。

### STEP 2 — Claudeサブエージェントにレビューさせる

STEP 1 と**同時に**走らせる。Agentツールで独立したレビュアーを起動し、
`templates/review-prompt.md` と設計図を読ませて、結果を `design/reviews/v1-claude.md` に保存させる。

- 会話の文脈を引き継がせない（設計図とレビュー観点だけを渡す）。文脈を知っていると忖度して指摘が甘くなる
- 設計を書いた本人の記憶を持ち込まないことが独立性の担保

### STEP 3 — 指摘を仕分けする

集まったファイルを読んで、重複を潰しながら以下に振り分ける。

| 分類 | 扱い |
|---|---|
| **MUST** | 無条件で v(N+1) に反映する |
| **SHOULD** かつ判断不要 | 反映する |
| **SHOULD** かつ判断が要る | **判断待ち**に回す（勝手に決めない） |
| **要判断** | **判断待ち**に回す |
| **CONSIDER** | 原則見送り。理由を明記して残す |
| **複数AIが同じ指摘** | 優先度を1段上げる（2本一致は信頼度が高い） |
| **AI同士で意見が割れた** | 潰さずに**両論併記で判断待ち**に上げる ← 一番大事な情報 |

**勝手に決めてはいけないもの（必ず判断待ちへ）**
予算・課金・料金設計 / スコープの増減 / リリース時期 / 使う外部サービスの選定 / 誰が運用するか / 取引先に見せる内容 / 法務・規約まわり

### STEP 4 — v(N+1) を作る

`templates/next-version.md` の骨格に沿って書く。**前の版は絶対に上書きしない**（差分を追えなくなる）。

必須セクション:
1. `## ⚠️ 判断待ち` — ユーザーが答えるまで着手不可。表形式。空でもセクションは残す
2. `## ✅ 今回反映した改善` — 何をどの指摘で直したか
3. `## ❌ 見送り` — 理由付き
4. `## 🔒 維持する設計判断` — 次の版で消さないもの
5. 以下、設計本文

### STEP 5 — ユーザーに報告する

- 判断待ちが**1件でもあれば、そこで止めて聞く**。答えが出るまで次の周に進まない
- 報告は「MUST何件反映 / 判断待ち何件 / 見送り何件」のサマリー + 判断待ちの中身
- **スキップされた外部AIがあれば、その理由（利用上限・未認証など）も一緒に伝える**
- 回答をもらったら `design/decisions.md` に追記してから v(N+1) を確定させる

### STEP 6 — 収束するまで回し続ける（版数の上限なし）

**2周連続で新規の指摘が出なくなるまで、何周でも繰り返す。** v3・v4・v5・v6・v7…と上限は設けない。
1周やって終わりにしない。**STEP 1 に戻るのが既定の動作。**

#### 打ち止め条件（これだけ）

> **新規の MUST と SHOULD が両方とも0件の周が、2回連続で発生した。**

- CONSIDER は打ち止め判定に**含めない**（細かい提案は永遠に出続けるため）
- **「新規」の判定が肝。** 既出の指摘が言い換えで再登場したものは新規に数えない。
  `design/decisions.md` に既出リストを持ち、毎周そこと突き合わせる。これをやらないと永久に収束しない
- 打ち止め条件を満たしていないのに勝手にやめない。**「もう十分では」という主観で止めるのは禁止**

#### 周回の進め方

| 状況 | 動作 |
|---|---|
| 判断待ちが**0件** | **確認を取らずに次の周へ自動で進む**（v3→v4→v5…と走り切る） |
| 判断待ちが**1件以上** | そこで一時停止して利用者に聞く。回答後に再開 |
| 新規MUST/SHOULDが0件（1回目） | まだ止めない。もう一周回して2回連続かを確かめる |
| 新規MUST/SHOULDが0件（2回連続） | **打ち止め。** 最終版を提示して完了報告 |
| 外部AIが上限でスキップされた周 | その周は判定に使わない（指摘が減ったのは上限のせいかもしれないため）。上限回復後に回し直す |

#### 毎周やること

`design/decisions.md` に1行ずつ追記していく：

```
| 周 | 版 | 新規MUST | 新規SHOULD | 判断待ち | 参加AI | 備考 |
|---|---|---|---|---|---|---|
| 1 | v2 | 3 | 5 | 2 | GPT/Gemini/Claude | |
| 2 | v3 | 1 | 2 | 0 | GPT/Gemini/Claude | |
| 3 | v4 | 0 | 0 | 0 | GPT/Gemini/Claude | 1回目のゼロ |
| 4 | v5 | 0 | 0 | 0 | GPT/Gemini/Claude | 2回連続 → 打ち止め |
```

- 10周を超えても収束しない場合は、**止めずに**「10周回っても新規指摘が出続けている。設計の粒度が大きすぎる可能性がある」と状況だけ報告し、利用者の指示を仰ぐ。勝手に打ち切らない
- 利用者が「ここまででいい」と言えば、条件を満たしていなくてもそこで止める

---

## 注意点

- **「良い点」を必ず拾う** — レビューテンプレに「残すべき設計判断」を書かせてある。これを無視すると、次の版で良い部分を消してしまう事故が起きる
- **外部AIはコードを読んでいない** — 渡したMDだけで判断している。実装と食い違う指摘は「前提が違う」として却下してよい。ただし却下理由は残す
- **レビュー結果は消さない** — `reviews/` に全部残す。「なぜこう決めたか」を後から追える資産になる
- 出力が空 or 極端に短いときは失敗を疑う。**「0件でした」を合格と扱わない**

## 外部AIが動かないときの対処

| 症状 | 原因と対処 |
|---|---|
| ChatGPTがスキップされる（未認証） | `codex login` を実行。無料プランでは利用できない場合がある |
| ChatGPTがスキップされる（利用上限） | プランの上限。数時間待つか、その系統なしで進める |
| Geminiがスキップされる（未認証） | **ターミナル.appで** `gemini` を起動 →「Use Gemini API Key」を選択 → [AI Studio](https://aistudio.google.com/apikey) のキーを入力。※個人向けOAuth（Sign in with Google）は2026-06-18に廃止済み |
| Geminiが「not running in a trusted directory」 | スクリプトは `--skip-trust` を付けているので通常は起きない。手動実行時は同フラグを付ける |
| 両方スキップされる | Claudeサブエージェント単独で続行する。ユーザーにその旨を伝える |
````

---

## ファイル2: `~/.claude/skills/design-review/templates/review-prompt.md`

````markdown
あなたは経験豊富なソフトウェア設計レビュアーです。
以下の設計図をレビューし、指定のフォーマットで日本語で回答してください。

## 前提

- この設計は個人〜小規模チームで開発・運用されます。大企業向けの過剰な体制論は不要です。
- 実装コードは渡されていません。設計図に書かれている内容だけで判断してください。
- 「設計図に書かれていないから穴だ」と決めつける前に、それが本当にこの規模で必要かを考えてください。

## レビュー観点

1. **前提の穴** — 暗黙のうちに正しいと仮定していて、崩れると設計全体が壊れるもの
2. **抜けている要件** — エラー時・空データ時・同時操作時・権限まわりなど、書かれていない分岐
3. **技術的リスク** — 性能・スケール・外部サービス依存・データ整合性
4. **運用とコスト** — 誰が・いつ・どうやって回すのか。ランニングコストは現実的か
5. **過剰設計** — この規模で不要なもの。作らなくていいものを指摘するのも重要な仕事です
6. **法務・コンプライアンス** — 個人情報、外部APIの利用規約、AI生成物の扱い（該当する場合のみ）

## 出力フォーマット（この見出し構成を厳守）

### MUST（このまま進めると事故る。必ず直すべき）
- **見出し** — 何が問題か / なぜ事故るか / どう直すか（具体的に）

### SHOULD（直したほうがいい。ただし致命的ではない）
- **見出し** — 内容 / 直し方

### CONSIDER（検討の余地はあるが、今やらなくてもいい）
- **見出し** — 内容

### 要判断（設計者本人が決めるしかないこと）
※ 予算・スコープ・リリース時期・外部サービスの選定・運用体制・法務など、
※ 技術的な正解が存在せず、本人の意思決定が必要なものだけをここに書いてください。
- **質問** — なぜ決める必要があるか / 選択肢A / 選択肢B / あなたの推奨

### 良い点（この版で維持すべき設計判断）
- **見出し** — なぜ良いか
※ 次の版でうっかり削られると困る判断を必ず挙げてください。

---

## 重要な指示

- 該当なしのセクションは「なし」と書いて、見出しは必ず残してください
- MUSTは本当に事故るものだけ。数を稼ぐために格上げしないでください
- 「一般論としてこうすべき」ではなく、**この設計図の具体的な記述を引用して**指摘してください
- 設計図の書き直し全文は不要です。指摘だけを返してください

---

# レビュー対象の設計図

````

> **注意**: このファイルの末尾は「# レビュー対象の設計図」の行と空行で終わります。
> この後ろにスクリプトが設計図本文を連結するため、余計な内容を足さないでください。

---

## ファイル3: `~/.claude/skills/design-review/templates/next-version.md`

````markdown
# <設計タイトル> v<N>

> 前版: `v<N-1>.md` ／ レビュー: `reviews/v<N-1>-gpt.md` `-gemini.md` `-claude.md` ／ 更新: YYYY-MM-DD

---

## ⚠️ 判断待ち（回答があるまで着手不可）

| # | 質問 | 背景・なぜ決める必要があるか | 選択肢 | 指摘元 | 推奨 |
|---|---|---|---|---|---|
| Q1 |  |  | A: <br>B:  | GPT/Gemini/Claude |  |

※ AI同士で意見が割れたものは、潰さずにここへ両論併記で上げる。
※ 空の場合も見出しは残し、「今回はなし」と書く。

---

## ✅ 今回反映した改善

| # | 内容 | 分類 | 指摘元 | 変更箇所 |
|---|---|---|---|---|
| 1 |  | MUST | GPT+Gemini（2本一致） | 「〇〇」の節 |

---

## ❌ 見送り（理由付き）

| # | 指摘 | 見送り理由 | 指摘元 |
|---|---|---|---|
| 1 |  | この規模では過剰／前提が実装と異なる／〇〇の後で判断する | Claude |

※ 「なんとなく」で見送らない。理由は必ず一文で書く。

---

## 🔒 維持する設計判断（次の版で消さないこと）

| 項目 | なぜ維持するか |
|---|---|
|  |  |

---

# 設計本文

<!-- ここから下が実際の設計。前版から変更のない部分もそのまま丸ごと載せる（差分だけにしない） -->
````

---

## ファイル4: `~/.claude/skills/design-review/scripts/run-external-reviews.sh`

**作成後に `chmod +x` を実行してください。**

````bash
#!/usr/bin/env bash
# 設計図MDを外部AI2系統（ChatGPT / Gemini）に並列で投げてレビューを取得する。
#
# usage: run-external-reviews.sh <設計図パス> <出力先ディレクトリ> <版ラベル>
# 例:    run-external-reviews.sh design/v1.md design/reviews v1
#
# 環境変数:
#   REVIEW_TIMEOUT  … 1系統あたりの制限時間（秒／既定 600）
#   REVIEW_SKIP     … 使わない系統をカンマ区切りで指定（例: REVIEW_SKIP=gemini）
#
# 設計方針:
#   - 未インストール／未ログイン／利用上限／時間切れ は「失敗」ではなく【スキップ】として扱い、他を止めない
#   - 1系統も取れなくても異常終了しない（Claudeサブエージェント側で続行できるため）
#   - ただしスキップした事実は必ず表示する。黙って握りつぶさない

set -uo pipefail

DESIGN="${1:?設計図のパスを指定してください}"
OUTDIR="${2:?出力先ディレクトリを指定してください}"
LABEL="${3:?版ラベル（例: v1）を指定してください}"

TIMEOUT="${REVIEW_TIMEOUT:-600}"
SKIP_LIST="${REVIEW_SKIP:-}"
MIN_CHARS=200

SKILL_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
PROMPT_TEMPLATE="$SKILL_DIR/templates/review-prompt.md"

[[ -f "$DESIGN" ]] || { echo "❌ 設計図が見つかりません: $DESIGN" >&2; exit 1; }
[[ -f "$PROMPT_TEMPLATE" ]] || { echo "❌ レビュープロンプトが見つかりません: $PROMPT_TEMPLATE" >&2; exit 1; }

mkdir -p "$OUTDIR"
WORK="$(mktemp -d)"
trap 'rm -rf "$WORK"' EXIT

# レビュー観点 + 設計図本文 を1本のプロンプトに結合
COMBINED="$WORK/prompt.md"
cat "$PROMPT_TEMPLATE" > "$COMBINED"
printf '\n```markdown\n' >> "$COMBINED"
cat "$DESIGN" >> "$COMBINED"
printf '\n```\n' >> "$COMBINED"

CHARS=$(wc -c < "$COMBINED" | tr -d ' ')
echo "📄 対象: ${DESIGN}（プロンプト計 ${CHARS} 文字）"
echo "⏱  1系統あたりの制限時間: ${TIMEOUT}秒"
echo "📤 ChatGPT と Gemini に並列で投げます…"
echo

GPT_OUT="$OUTDIR/${LABEL}-gpt.md"
GEM_OUT="$OUTDIR/${LABEL}-gemini.md"
S_GPT="$WORK/gpt.status"; L_GPT="$WORK/gpt.log"
S_GEM="$WORK/gem.status"; L_GEM="$WORK/gem.log"

# 指定された系統をスキップするか
is_skipped() {
  [[ ",$SKIP_LIST," == *",$1,"* ]]
}

# 指定PIDを見張り、制限時間を超えたら強制終了する。終了コードを返す（時間切れは137）
guard() {
  local pid="$1"
  ( sleep "$TIMEOUT"; kill -9 "$pid" 2>/dev/null ) &
  local watchdog=$!
  wait "$pid" 2>/dev/null
  local rc=$?
  kill "$watchdog" 2>/dev/null
  wait "$watchdog" 2>/dev/null
  return "$rc"
}

# 終了コードとログから、何が起きたのかを判定する
classify() {
  local rc="$1" log="$2" out="$3" statusfile="$4"

  if [[ "$rc" -eq 137 || "$rc" -eq 143 ]]; then
    echo "SKIP_TIMEOUT" > "$statusfile"; return
  fi

  local size=0
  [[ -f "$out" ]] && size=$(wc -c < "$out" | tr -d ' ')

  # 判定に使うテキスト。出力が正常な長さのときは本文を見ない（設計図の中身で誤判定するため）
  local text
  text="$(tr '[:upper:]' '[:lower:]' < "$log" 2>/dev/null)"
  if [[ "$size" -lt "$MIN_CHARS" && -f "$out" ]]; then
    text="$text $(tr '[:upper:]' '[:lower:]' < "$out" 2>/dev/null)"
  fi

  # 利用上限・レート制限
  if grep -qE 'rate limit|rate_limit|quota|resource_exhausted|usage limit|429|too many requests|exceeded your|limit reached|out of credit|insufficient_quota|upgrade your plan' <<< "$text"; then
    echo "SKIP_QUOTA" > "$statusfile"; return
  fi

  # 未ログイン・認証エラー
  if grep -qE 'not logged in|please log in|sign in|authenticate|auth method|no longer supported|api key|401|403|unauthorized|invalid_api_key|permission denied' <<< "$text"; then
    echo "SKIP_AUTH" > "$statusfile"; return
  fi

  if [[ "$rc" -ne 0 ]]; then
    echo "FAIL_OTHER" > "$statusfile"; return
  fi
  if [[ "$size" -lt "$MIN_CHARS" ]]; then
    echo "WARN_SHORT" > "$statusfile"; return
  fi
  echo "OK" > "$statusfile"
}

# ---- ChatGPT (codex CLI) ----
run_gpt() {
  if is_skipped gpt || is_skipped chatgpt || is_skipped codex; then
    echo "SKIP_BY_USER" > "$S_GPT"; return
  fi
  if ! command -v codex >/dev/null 2>&1; then
    echo "SKIP_NOT_INSTALLED" > "$S_GPT"; return
  fi
  # --skip-git-repo-check: リポジトリ外でも動かす / --ephemeral: セッションを残さない
  codex exec --skip-git-repo-check --ephemeral -o "$GPT_OUT" - < "$COMBINED" > "$L_GPT" 2>&1 &
  guard $!
  classify $? "$L_GPT" "$GPT_OUT" "$S_GPT"
}

# ---- Gemini (gemini CLI) ----
run_gemini() {
  if is_skipped gemini; then
    echo "SKIP_BY_USER" > "$S_GEM"; return
  fi
  if ! command -v gemini >/dev/null 2>&1; then
    echo "SKIP_NOT_INSTALLED" > "$S_GEM"; return
  fi
  # --skip-trust: 非対話環境では「信頼済みフォルダ」判定で弾かれるため必須
  gemini --skip-trust -p "$(cat "$COMBINED")" < /dev/null > "$GEM_OUT" 2> "$L_GEM" &
  guard $!
  classify $? "$L_GEM" "$GEM_OUT" "$S_GEM"
}

run_gpt &    PID_GPT=$!
run_gemini & PID_GEM=$!
wait "$PID_GPT" "$PID_GEM"

# ---- 結果表示 ----
OK_COUNT=0
SKIPPED_STR=""

# 変数の直後に全角文字を置くと bash が変数名として解釈して落ちる。必ず ${var} で囲むこと。
note_skip() {
  if [[ -z "$SKIPPED_STR" ]]; then SKIPPED_STR="$1"; else SKIPPED_STR="${SKIPPED_STR} / $1"; fi
}

report() {
  local name="$1" file="$2" statusfile="$3" cli="$4"
  local status; status="$(cat "$statusfile" 2>/dev/null || echo 'FAIL_OTHER')"
  local size=0
  [[ -f "$file" ]] && size=$(wc -c < "$file" | tr -d ' ')

  case "$status" in
    OK)
      echo "✅ ${name}: ${file}（${size} 文字）"
      OK_COUNT=$((OK_COUNT+1)) ;;
    SKIP_QUOTA)
      echo "⏭️  ${name}: 【スキップ】利用上限・レート制限に達しています"
      echo "      → 上限が回復してから再実行するか、この系統なしで進めてください"
      note_skip "${name}（利用上限）"; rm -f "$file" ;;
    SKIP_AUTH)
      echo "⏭️  ${name}: 【スキップ】未ログイン、または認証が通っていません"
      echo "      → ターミナルで \`${cli}\` を起動してログインしてください"
      note_skip "${name}（未認証）"; rm -f "$file" ;;
    SKIP_TIMEOUT)
      echo "⏭️  ${name}: 【スキップ】${TIMEOUT}秒を超えたため打ち切りました"
      echo "      → REVIEW_TIMEOUT=1200 のように延ばして再実行できます"
      note_skip "${name}（時間切れ）"; rm -f "$file" ;;
    SKIP_NOT_INSTALLED)
      echo "⏭️  ${name}: 【スキップ】${cli} コマンドが入っていません"
      note_skip "${name}（未インストール）" ;;
    SKIP_BY_USER)
      echo "⏭️  ${name}: 【スキップ】REVIEW_SKIP で除外されています"
      note_skip "${name}（手動除外）" ;;
    WARN_SHORT)
      echo "⚠️  ${name}: 出力が短すぎます（${size} 文字）。中身を確認してください → ${file}"
      note_skip "${name}（出力不足）" ;;
    *)
      echo "❌ ${name}: 失敗しました（想定外のエラー）"
      note_skip "${name}（エラー）"
      [[ "$size" -eq 0 ]] && rm -f "$file" ;;
  esac
}

echo "=== 結果 ==="
report "ChatGPT" "$GPT_OUT" "$S_GPT" "codex"
report "Gemini " "$GEM_OUT" "$S_GEM" "gemini"
echo
echo "外部AI 取得成功: ${OK_COUNT} 系統"
if [[ -n "$SKIPPED_STR" ]]; then
  echo "スキップ: ${SKIPPED_STR}"
  echo "※ スキップした系統は、そのままにせず必ずユーザーに報告すること。"
fi
if [[ "$OK_COUNT" -eq 0 ]]; then
  echo "※ 外部AIは全滅ですが、Claudeサブエージェントのレビューだけで続行できます。"
fi
exit 0
````

---

## 導入後にやること（Claude向け）

### 1. 実行権限の付与と構文チェック

```bash
chmod +x ~/.claude/skills/design-review/scripts/run-external-reviews.sh
bash -n ~/.claude/skills/design-review/scripts/run-external-reviews.sh
```

### 2. 各AIが使える状態か確認する

```bash
command -v codex  || echo "codex 未インストール"
command -v gemini || echo "gemini 未インストール"
command -v codex  && codex login status 2>&1 | head -2
command -v gemini && gemini --skip-trust -p "Reply with exactly: OK" < /dev/null 2>&1 | tail -3
```

### 3. 未インストールのCLIがあれば、導入を手伝う

**「入っていませんでした」で終わらせない。** インストールするかどうかを利用者に尋ね、
希望されたらインストールまで実行する。

まず前提を確認する：

```bash
node -v && npm -v      # 無ければ Node.js の導入から案内する（https://nodejs.org）
```

利用者の同意を得てから実行する：

```bash
npm install -g @openai/codex        # ChatGPT用
npm install -g @google/gemini-cli   # Gemini用
```

インストール後、**ログイン作業は利用者本人にターミナル.appでやってもらう**
（Claude Code のシェルは対話端末を持たないため、認証ウィザードが起動しない）：

- **ChatGPT**: ターミナルで `codex login`
  ※ ChatGPT無料プランでは利用できないか、すぐ上限に達する旨を先に伝えること
- **Gemini**: ターミナルで `gemini` →「2. Use Gemini API Key」を選択 → [AI Studio](https://aistudio.google.com/apikey) で発行したキーを貼り付け
  ※「1. Sign in with Google」は2026-06-18に個人向けが廃止済み。選ばせないこと

利用者が「入れなくていい」と言えば、そのまま進める。**無理強いしない。**
外部AIがゼロでも Claude サブエージェント単独でスキルは動作する。

### 4. スキップ動作の確認（外部AIを呼ばないので無料・数秒）

適当なテスト用設計図を作り、両系統を除外して実行する：

```bash
REVIEW_SKIP=gpt,gemini ~/.claude/skills/design-review/scripts/run-external-reviews.sh <テスト用MD> /tmp/rv-test v1
```

「【スキップ】REVIEW_SKIP で除外されています」が2件出て、
`外部AI 取得成功: 0 系統` と表示され、**エラーで落ちずに終了する**ことを確認する。

### 5. 利用者への報告

以下を伝えること：

- スキルの導入が完了したこと、**「マルチAIレビューして」で起動する**こと
- **どのAIが現在使える状態か**（Claudeは常に可。ChatGPT/Geminiは上の確認結果に基づく）
- 使えないAIがある場合、その理由と、導入したいなら手伝える旨
- **3つ揃っていなくてもスキルは動く**こと
- **1周で終わらず、新規の指摘が出ない周が2回連続するまで自動で回り続ける**こと（v3・v4・v5…と進む）。
  途中で止めたいときは「ここまでにして」と言えば止まること

### 注意

- 既に `~/.claude/skills/design-review/` が存在する場合は、**上書きする前に利用者に確認する**
- 各AIのセットアップ（`codex login` や Gemini のAPIキー入力）は**対話端末が必要なため、Claude Code からは実行できない**。利用者本人にターミナルで実行してもらうこと
