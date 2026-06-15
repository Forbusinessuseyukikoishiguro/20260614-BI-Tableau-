# うさうさ育成ごっち v3.2.0 ドキュメント一式

対象成果物：`【PRO】usausa-test-100 (20260615)_100plus.html`（単一HTML / オフライン）
版数：v3.2.0（2026-06-15）／ 発行：うさうさ研修工房
記載方針：**本書はすべて添付の実装コードに基づく事実記述（嘘つかず🐰）**。コードに存在しない機能・数値は記載しない。

> 本ファイルは7文書を1つにまとめたもの。各文書は「━━ 文書N ━━」で区切ってあり、章ごとにコピーして別Wordへ貼れば分冊にできる。

### 収録文書
1. 要件定義書
2. 詳細設計書
3. アーキテクチャ仕様書
4. 実装仕様書
5. テスト仕様書
6. 完全再現実装仕様書
7. カスタマイズ方法

---
---

# ━━ 文書1 ━━ 要件定義書
> 保存ファイル名案：`01_要件定義書_usausa_v3.2.docx`

## 1.1 目的
データ分析基礎・BI・Tableau の用語と概念を、ゲーム要素と間隔反復で**継続的に反復学習**できる自習教材を、配布容易な単一HTMLとして提供する。

## 1.2 想定利用者・利用環境
- 研修受講者、独学者、復習したい実務者。
- モダンブラウザ（Chrome / Edge / Safari / Firefox）。インストール・サーバー・ビルド不要。
- 学習・育成はオフラインで完結。外部通信は「参考リンク先の閲覧」「AI問題生成」のときのみ。

## 1.3 機能要件
| ID | 要件 | 充足機能 |
|---|---|---|
| FR-01 | 4択クイズを出題・採点する | まなぶ／100問テスト／10問チャレンジ |
| FR-02 | 約100問を内蔵する | 内蔵問題配列 `QS`（基本セット＋追加6問） |
| FR-03 | 問題を前後に行き来できる | 100問テストの「まえ／つぎ」＋問題パレット |
| FR-04 | 正誤（○×）を後から見直せる | パレットの色分け＋回答済みのレビュー表示＋結果画面の誤答一覧 |
| FR-05 | 答えに公式・関連・論文の参考リンクを出す | `resolveRefs` / `renderRefs`（公式/関連/論文） |
| FR-06 | 途中の一時保存ができる | 回答ごとの自動保存＋「💾 一時保存」ボタン |
| FR-07 | 閉じても翌日続きから再開できる | localStorage への状態保存＋起動時の再開導線 |
| FR-08 | 学習記録を出力できる | Markdown / TXT / CSV / Excel / 印刷(PDF) |
| FR-09 | 問題を追加できる | 手動追加・JSON入出力・AI生成（任意） |
| FR-10 | 育成（進化）で継続を促す | XP・段階進化・首飾り収集 |
| FR-11 | 忘却曲線に沿って再出題する | 間隔反復（誤答=翌日 / 正答=1→3→7→16日） |

## 1.4 非機能要件
- **可搬性**：単一HTML、外部ライブラリ依存なし。
- **永続性**：進行は端末・ブラウザの localStorage に保存（後述の制約あり）。
- **正確性**：参考リンクは一次情報への入口とし、断定は最小限（「嘘つかず」方針）。
- **アクセシビリティ**：`prefers-reduced-motion` でアニメ抑制。`aria-pressed` 等の付与。
- **安全性**：APIキーは利用者持ち・localStorage 保存・送信先はAIプロバイダのみ。

## 1.5 制約・前提（事実）
- 進行データは**その端末・そのブラウザの localStorage 依存**。別端末・別ブラウザ・シークレット・キャッシュ削除では引き継がれない。
- 内蔵問題は基礎レベル中心。資格試験そのものの代替ではなく反復・土台固め用。
- AI生成は任意機能であり、利用料は各プロバイダの課金に従う。

## 1.6 スコープ外
- サーバー同期・複数端末同期・ユーザー認証・ランキング共有・自動採点の外部送信。

---
---

# ━━ 文書2 ━━ 詳細設計書
> 保存ファイル名案：`02_詳細設計書_usausa_v3.2.docx`

## 2.1 画面（パネル）構成
ハードウェア風の本体（device）下に、単一の `#panel` を機能ごとに描き替えるSPA構成。

| 操作ボタン | 関数 | 内容 |
|---|---|---|
| 📚 まなぶ | `panelLearn` | ランダム1問出題・採点・解説・参考リンク |
| 🍽️ ごはん | `panelFeed` | 草/人参でまんぷく・きげん回復 |
| 📊 ようす | `panelStatus` | 段階・XP・統計・首飾り・設定・リセット |
| 📝 100問テスト | `panelTest` | 100問テスト本体（中心機能） |
| 🎮 10問 | `panelChallenge` | データ分析10問チャレンジ＋首飾り |
| 📥 きろく | `panelRecord` | メモ・スナップショット・各形式出力 |
| 🧩 もんだい | `panelQuestions` | 問題追加（手動/AI）・JSON入出力 |
| （起動時） | `panelWelcome` | 説明・再開導線・復習件数 |

## 2.2 状態モデル（`S`）
`DEFAULT` を基底に localStorage から復元。主なフィールド：

| フィールド | 型 | 用途 |
|---|---|---|
| full, mood | number(0–100) | まんぷく・きげん |
| xp | number | 知識XP（段階の基準） |
| grass, carrot | number | 通貨（草・人参） |
| learned, correctTotal, streak, best | number | 学習統計 |
| born, last | number(ms) | 開始日時・最終保存 |
| sound | boolean | 効果音ON/OFF |
| history | array | 学習履歴（最大400件） |
| necklaces, equipped | array,string | 首飾り所持・装着中 |
| challengeBest | number | チャレンジ最高スコア |
| memo | string | 一時メモ |
| customQ | array | 追加問題 |
| srs | object | 間隔反復の状態（問題文→{box,due,graduated}） |
| test | object\|null | 100問テストのスナップショット |

## 2.3 段階・通貨・報酬設計（実装値）
- 進化段階（XP閾値）：たまご `0` → ベビーうさ `30` → こうさぎ `120` → みならいうさ `300` → うさうさ先生🐰 `600`。
- まなぶ報酬：やさしい正解→🌿+1・XP+10／むずかしい正解→🥕+1・XP+15／不正解→XP+3。3連続正解ごとに🥕+1。
- ごはん：草＝まんぷく+8/きげん+4。人参＝まんぷく+20/きげん+12/XP+5。
- チャレンジ：正解XP+6／不正解XP+2。100問テスト：正解XP+4／不正解XP+1。
- 首飾り判定 `neckForScore`：10点→👑gold／9点→💧aqua／7点以上→🌸sakura／それ未満→🎀ribbon。
- 時間減衰：`tick` が12秒ごとに（段階>たまごのとき）まんぷく−2、まんぷく<30できげん−1。離席時は10分ごとにまんぷく−1（上限40）。

## 2.4 間隔反復（SRS）設計
- 単位：問題文文字列をキーに `S.srs[q] = {box, due, graduated}`。
- 間隔：`SRS_INTERVALS=[1,3,7,16]`（日）。
- 正解：box+1。box が4を超えたら `graduated=true`（再出題対象外）。それ以外は `due = 当日0時 + 間隔[box-1]日`。
- 不正解：box=1、`due=当日0時+1日`（翌日）。
- 出題優先度（`pickQuestion`）：期限到来(due≤now) > 未出題 > 未卒業を期限順 > ランダム。

## 2.5 100問テスト設計
- スナップショット `S.test = {qs, ans, ord, i, done, startedAt, savedAt}`。
  - `qs`：出題する問題オブジェクト最大`TEST_SIZE(=100)`件（`ALLQ()`をシャッフルして抜粋）。
  - `ans[k]`：各問の回答（選択肢インデックス、未回答は null）。回答後はロック。
  - `ord[k]`：各問の選択肢表示順（生成時にシャッフルし固定）→再訪時もレイアウト不変。
  - `i`：現在位置、`done`：完了フラグ。
- ナビゲーション：まえ/つぎ、10×10パレット（緑=正解/赤=不正解/灰=未回答、タップでジャンプ）。
- 永続：回答・移動・保存のたびに `save()`。起動時に未完了テストがあれば再開導線（welcome＋トースト）。

## 2.6 参考リンク解決設計
- `REF_RULES`：正規表現 → 参考リンク配列（`{l:表示名, u:URL, k:official|related|academic}`）。
- `resolveRefs(Q)`：問題文にマッチした規則のリンクを集約、重複URL除去、公式が無ければカテゴリ既定の公式を補完、優先度（公式→関連→論文）でソート、最大4件。
- URL生成補助：`WK`（Wikipedia日本語）、`TBH`（Tableau公式ヘルプ）、`gs()`（サイト内Google検索）、`sch()`（Google Scholar検索）。

## 2.7 出力設計（きろく）
`buildMarkdown / buildTxt / buildCsv / buildXls / buildPrint` がサマリー＋履歴＋メモを各形式へ整形し、`downloadBlob` で保存。CSV/XLSはBOM付与で文字化け対策。印刷はPDF保存用に `#printArea` を差し替えて `window.print()`。

---
---

# ━━ 文書3 ━━ アーキテクチャ仕様書
> 保存ファイル名案：`03_アーキテクチャ仕様書_usausa_v3.2.docx`

## 3.1 全体像
- 形態：**単一HTMLのクライアントサイドSPA**（HTML + CSS + Vanilla JS）。
- 外部依存：なし（フレームワーク・CDN・フォント外部読込なし）。
- 状態：単一グローバル `S` を localStorage に直列化して永続化。
- 描画：`#panel` を関数が `innerHTML` で再描画する「画面=関数」方式。
- 入力安全：全表示文字列は `e()`（HTMLエスケープ）を通す。

## 3.2 レイヤ構成（論理）
```
┌──────────────── UI層（DOM / CSS） ────────────────┐
│ device(本体), #panel(画面), #toast, #printArea           │
└───────────────▲───────────────────────────────────┘
                │ render*/panel* が innerHTML を生成
┌───────────────┴── プレゼンテーション層（描画関数） ──┐
│ renderPet/renderStats/renderClock/renderExp/renderRefs   │
│ panelLearn/panelTest/panelChallenge/panelStatus/...      │
└───────────────▲───────────────────────────────────┘
                │ 参照・更新
┌───────────────┴── ドメイン層（ロジック） ───────────┐
│ pickQuestion, srsUpdate, resolveRefs, startTest,         │
│ answer*, finishChallenge, summaryObj, build*(出力)       │
└───────────────▲───────────────────────────────────┘
                │ 読み書き
┌───────────────┴── 永続化層 ───────────────────────┐
│ save/load(localStorage: usausa_gotchi_v3 ほか)           │
│ 任意: fetch(OpenAI/Anthropic)  ← AI生成時のみ            │
└───────────────────────────────────────────────────┘
```

## 3.3 データフロー（例：100問テスト回答）
1. ユーザーが選択肢をタップ → `answerTest(oi)`。
2. `S.test.ans[i]=oi` 記録 → `srsUpdate` でSRS更新 → 統計/XP加算。
3. `logHist` で履歴追加 → `save()` で永続化。
4. `renderTestQ()` 再描画（レビュー表示＋解説＋`renderRefs`＋ナビ）。

## 3.4 永続化スキーマ（localStorage キー）
| キー | 内容 | 書込関数 |
|---|---|---|
| `usausa_gotchi_v3` | 本体状態 `S` 全体 | `save()` |
| `usausa_gotchi_snapshot` | 状態スナップショット `{t,s}` | きろくの「状態を一時保存」 |
| `usausa_api_provider` | AIプロバイダ（openai/anthropic） | もんだい→AI設定 |
| `usausa_api_model` | モデル名 | 同上 |
| `usausa_api_key` | APIキー | 同上 |

## 3.5 外部インターフェース
- AI生成：`POST https://api.openai.com/v1/chat/completions`（OpenAI）または `POST https://api.anthropic.com/v1/messages`（Anthropic、ブラウザ直アクセスヘッダ付与）。任意・キー未設定なら未使用。
- 参考リンク：外部サイトへの遷移（Wikipedia / Tableau Help / Microsoft Learn / Google検索・Scholar 等）を新規タブで開く。

## 3.6 タイマー・ライフサイクル
- `setInterval(renderClock, 20s)`、`setInterval(tick, 12s)`。
- `beforeunload` で `save()`。

---
---

# ━━ 文書4 ━━ 実装仕様書
> 保存ファイル名案：`04_実装仕様書_usausa_v3.2.docx`

## 4.1 モジュール（機能ブロック）と主関数
| ブロック | 主な関数 |
|---|---|
| 共通 | `e`（エスケープ）、`clamp`、`shuffleArr`、`fmtTs`、`dateStamp` |
| 描画(本体) | `petSVG`、`neckEl`、`sparkles`、`renderPet`、`renderStats`、`renderClock`、`floatEmoji`、`hop`、`flash`、`toast`、`beep` |
| 状態 | `save`、`load`、`logHist` |
| SRS | `startOfDay`、`isSeen`、`isDue`、`srsUpdate`、`dueCount`、`dueBadge` |
| タブ | `setTab` |
| まなぶ | `pickQuestion`、`renderExp`、`renderChoices`、`markChoices`、`panelLearn`、`answerLearn` |
| ごはん | `panelFeed`、`feed` |
| チャレンジ | `daPool`、`panelChallenge`、`startChallenge`、`renderChallengeQ`、`answerChallenge`、`finishChallenge`、`bindNeckGrid` |
| 100問テスト | `testExists`、`testAnsweredCount`、`testCorrectCount`、`panelTest`、`testStartScreen`、`startTest`、`renderTestProgressHead`、`renderTestChoices`、`renderTestReviewChoices`、`renderTestNav`、`renderTestPalette`、`renderTestLegend`、`renderTestQ`、`answerTest`、`bindTestNav`、`renderTestResult` |
| ようす | `panelStatus` |
| 参考リンク | `R`、`gs`、`sch`、`resolveRefs`、`renderRefs`（データ：`WK`,`TBH`,`CAT_FALLBACK`,`REF_RULES`） |
| きろく/出力 | `summaryObj`、`kindLabel`、`downloadBlob`、`buildMarkdown`、`buildTxt`、`buildCsv`、`buildXls`、`buildPrint`、`doPrint`、`panelRecord` |
| もんだい/AI | `apiCfg`、`catCounts`、`panelQuestions`、`renderGenPreview`、`validateQ`、`genPrompt`、`parseGenerated`、`generateAI` |
| 起動/ループ | `panelWelcome`、`tick`、起動時のイベント結線 |

## 4.2 主要関数の契約（入出力・副作用）
- `load()`→`S`：DEFAULTへ復元、配列/型の健全化、`test`の妥当性検査（`qs`/`ans`が配列でなければnull化、`ord`欠落時は自然順で補完）、離席減衰を適用。
- `save()`：`S.last`更新後 `usausa_gotchi_v3` へ直列化（try/catch）。
- `srsUpdate(q, correct)`：SRS状態を更新（副作用：`S.srs`）。
- `pickQuestion()`→`Q`：出題優先度に従い1問返す。
- `answerLearn / answerChallenge / answerTest`：採点・SRS更新・統計/XP/通貨更新・`logHist`・`save`・再描画。
- `resolveRefs(Q)`→`refs[]`、`renderRefs(Q)`→HTML文字列（最大4件、`target="_blank" rel="noopener noreferrer"`）。
- `validateQ(o, cat)`→正規化済み問題 or null（不正データはnull）。
- `generateAI(cfg,cat,diff,n)`→Promise<問題[]>（プロバイダ別fetch、エラー時 throw）。

## 4.3 問題データ仕様
```
{ cat:"データ分析基礎"|"BI"|"Tableau", diff:"easy"|"hard",
  q:string, ch:[s,s,s,s], a:0..3,
  exp:{ def:string, hito:string, tatoe?:string, detail?:string },
  da:boolean }   // da=true は10問チャレンジ対象
```
全問題＝`ALLQ()`＝`QS.concat(S.customQ)`。`QS`は基本配列＋`QS.concat([...6問])`。

## 4.4 UI/スタイル方針
- CSS変数でブランド配色を集中管理（`--shell/--coral-*/--carrot/--grass/--gold` 等）。
- うさぎはSVGをJS文字列で生成（段階・気分・首飾りで分岐）。
- トースト通知、フロート絵文字、フラッシュ等の軽量アニメーション。

## 4.5 エラーハンドリング
- localStorage操作・JSON解析・fetchはtry/catchで握り、`toast`でユーザー通知。
- JSON取込・AI生成は `validateQ`/`parseGenerated` で不正要素を除外。

---
---

# ━━ 文書5 ━━ テスト仕様書
> 保存ファイル名案：`05_テスト仕様書_usausa_v3.2.docx`

## 5.1 テスト環境
- 対象：当該HTMLをローカル（file://）およびブラウザで開く。
- 主要ブラウザ：Chrome / Edge / Safari。必要に応じ Firefox。
- 事前条件：通常ウィンドウ（localStorage有効）。初期化確認時はリセット実施。

## 5.2 機能テストケース
| ID | 観点 | 手順 | 期待結果 |
|---|---|---|---|
| T-01 | 起動 | ファイルを開く | うさぎ描画・ようこそ画面・時計表示。エラーなし |
| T-02 | まなぶ採点 | 📚→選択 | 正誤判定、解説・参考リンク表示、XP/通貨が増加 |
| T-03 | 進化 | XPが閾値超過 | 段階名が次段階へ更新、演出表示 |
| T-04 | ごはん | 🍽️→草/人参 | まんぷく・きげんが規定値だけ回復、所持数減少 |
| T-05 | テスト開始 | 📝→スタート | 最大100問が出題、第1問表示 |
| T-06 | 前後ナビ | まえ/つぎ | 端で無効化、移動で問題が切替 |
| T-07 | パレット | マスをタップ | 当該問題へジャンプ、現在位置が強調 |
| T-08 | ○×反映 | 回答後にパレット確認 | 正解=緑/不正解=赤に変化 |
| T-09 | 見直し | 回答済みへ戻る | 正解=緑・誤答=赤で読み取り専用表示、参考リンクあり |
| T-10 | 参考リンク | リンクをクリック | 新規タブで該当一次情報が開く |
| T-11 | 一時保存 | 💾一時保存 | トースト表示、内容保持 |
| T-12 | 再開(同タブ) | 別タブへ→📝に戻る | 続きの問題位置から再開 |
| T-13 | 再開(再起動) | ブラウザ閉→再度開く | ようこそに再開カード、続きから可能 |
| T-14 | 結果 | 結果ボタン | 正答率・内訳・誤答一覧表示、誤答タップで該当問題へ |
| T-15 | チャレンジ | 🎮→10問完了 | スコアに応じ首飾り付与・装着 |
| T-16 | 記録出力 | 📥→各形式 | MD/TXT/CSV/XLS生成、印刷でPDF保存可 |
| T-17 | 手動追加 | 🧩→入力→追加 | 合計問題数が+1、テスト/まなぶに反映 |
| T-18 | JSON入出力 | 書出→読込 | 件数表示、取込後に反映 |
| T-19 | リセット | ようす→はじめから | 履歴・首飾り・メモ・カスタム・テストが初期化 |

## 5.3 異常系・境界
| ID | 観点 | 手順 | 期待結果 |
|---|---|---|---|
| E-01 | 不正JSON取込 | 壊れたJSONを読込 | 「読み込み失敗」トースト、状態破壊なし |
| E-02 | 不正問題要素 | ch3件のみ等を含むJSON | 当該要素を除外して取込 |
| E-03 | AIキー未設定 | 生成ボタン押下 | キー入力を促すトースト、内蔵問題で継続可 |
| E-04 | AI生成失敗 | 無効キーで生成 | エラーメッセージ表示、アプリ継続 |
| E-05 | 回答ロック | 回答済みを再タップ | 二重採点されない |
| E-06 | 既回答の再採点 | パレットで戻る | 統計/XPが再加算されない |
| E-07 | reduce-motion | OS設定有効で起動 | アニメーション抑制 |
| E-08 | localStorage不可 | 制限環境で操作 | 例外を握り、当該セッションは継続 |

## 5.4 回帰確認の必須3点（過去不具合の再発防止）
- R-01：ファイル末尾まで完全であること（`</script></body></html>` まで存在）。
- R-02：全操作ボタン（まなぶ/ごはん/ようす/100問テスト/10問/きろく/もんだい）が反応すること。
- R-03：起動直後にうさぎSVGと時計が表示されること（JS未完による無反応がないこと）。

## 5.5 合否基準
T-01〜T-19 と R-01〜R-03 がすべて期待結果どおりであれば合格。E系は致命的破壊が起きないことをもって合格とする。

---
---

# ━━ 文書6 ━━ 完全再現実装仕様書
> 保存ファイル名案：`06_完全再現実装仕様書_usausa_v3.2.docx`
> 目的：本書のみでゼロから同等アプリを再構築できる水準の定義。

## 6.1 グローバル定数（実装値）
| 定数 | 値 |
|---|---|
| `STAGES` 閾値 | egg=0, baby=30, young=120, prentice=300, master=600 |
| `NECKS` | ribbon🎀, sakura🌸, aqua💧, gold👑 |
| `NECK_ORDER` | [ribbon, sakura, aqua, gold] |
| `CATS` | ["データ分析基礎","BI","Tableau"] |
| `SRS_INTERVALS` | [1,3,7,16]（日） |
| `SRS_DAY` | 86400000（ms） |
| `TEST_SIZE` | 100 |
| 履歴上限 | 400件 |
| 離席減衰 | 10分ごとに full−1（上限40）、full<30 で mood減衰 |
| tick | 12秒ごと full−2、full<30 で mood−1（段階>egg時） |
| 時計更新 | 20秒ごと |

## 6.2 状態スキーマ `DEFAULT`
`{full:80, mood:80, xp:0, grass:0, carrot:0, learned:0, correctTotal:0, streak:0, best:0, born:now, last:now, sound:true, asked:[], history:[], necklaces:[], equipped:null, challengeBest:0, memo:"", customQ:[], srs:{}, test:null}`

## 6.3 問題スキーマ
6.3章＝文書4 4.3に同じ。`da:true` のみ10問チャレンジ対象。

## 6.4 SRSアルゴリズム（擬似コード）
```
isDue(q): srs[q] && !graduated && due<=now
srsUpdate(q, correct):
  r = srs[q] || {box:0}
  if correct:
    r.box += 1
    if r.box > 4: r.graduated=true; r.due=null
    else: r.due = startOfDay(now) + INTERVALS[r.box-1]*DAY
  else:
    r.box=1; r.graduated=false; r.due = startOfDay(now)+1*DAY
  srs[q]=r
pickQuestion(): due > unseen > (未卒業をdue昇順) > random
```

## 6.5 100問テストのライフサイクル
```
startTest():
  pool = shuffle(ALLQ())
  qs   = pool[0 .. min(100, pool.length))
  test = { qs, ans:[null...], ord:[shuffle([0,1,2,3]) ...],
           i:0, done:false, startedAt:now, savedAt:now }
answerTest(oi): 未回答時のみ ans[i]=oi → SRS/統計/XP → log → save → 再描画
パレット状態: ans[k]==null→灰 / ans[k]==qs[k].a→緑 / それ以外→赤
結果: 全問回答で done=true（履歴に完了記録）
再開: 起動時 test!=null && !done なら welcome に再開導線
```

## 6.6 参考リンク解決（擬似コード）
```
resolveRefs(Q):
  out=[]; for rule in REF_RULES: if rule.re.test(Q.q): out += rule.refs(URL重複除去)
  if out空: out += CAT_FALLBACK[Q.cat]
  else if outに official 無: out += CAT_FALLBACK[Q.cat]
  sort by kind(official<related<academic); return out[0..4]
URL生成: WK=ja.wikipedia.org/wiki/<語>, TBH=Tableau公式ヘルプ,
        gs(site,q)=Google site内検索, sch(q)=Google Scholar検索
表示: kind→ラベル{official:公式, related:関連, academic:論文}
```

## 6.7 報酬・経済モデル（再掲・実装値）
- まなぶ：easy正解 grass+1/xp+10、hard正解 carrot+1/xp+15、誤答 xp+3、3連続で carrot+1。
- ごはん：草 full+8/mood+4、人参 full+20/mood+12/xp+5。
- チャレンジ：xp +6/+2。テスト：xp +4/+1。
- 首飾り：score≥10 gold / ≥9 aqua / ≥7 sakura / else ribbon。

## 6.8 出力フォーマット
- Markdown/TXT：サマリー＋学びの要点（学習正解のhito、直近30）＋履歴（直近50）＋メモ。
- CSV/XLS：BOM付き、列＝日時/種別/カテゴリ/難易度/正誤/内容/ひとこと。
- 印刷：サマリー＋履歴（直近100）を `#printArea` に描画し `window.print()`。

## 6.9 起動シーケンス
```
S = load()
各ボタンに onclick を結線（learn/feed/status/test/chal/rec/q）
renderPet(); renderStats(); renderClock(); panelWelcome()
未完了テストあり→「続きがあります」トースト / 無→復習件数トースト
setInterval(renderClock,20s); setInterval(tick,12s)
beforeunload→save()
```

## 6.10 既知の制約（再現時も同等）
- 進行は端末/ブラウザ依存（localStorage）。
- テストの `qs` は全問題オブジェクトをスナップショット保持（再現性のため）。
- AI生成はブラウザ直fetch（CORS/キーは利用者責任）。

---
---

# ━━ 文書7 ━━ カスタマイズ方法
> 保存ファイル名案：`07_カスタマイズ方法_usausa_v3.2.docx`

> いずれもHTMLを直接編集。バックアップを取り、編集後はブラウザで再読込して確認すること。

## 7.1 問題を追加する（コード編集なし／推奨）
- アプリ内 **🧩 もんだい**：手動追加、または **JSON読み込み**。形式は文書4 4.3参照。
- JSONは配列で渡す。不正要素は自動スキップ。`cat:"データ分析基礎"` は10問チャレンジ対象になる。

## 7.2 内蔵問題を増やす（コード編集）
`QS=QS.concat([ ... ]);` のブロックに問題オブジェクトを追記。100問テストは自動的に最大100問を出題（母数が増えれば多様性が増す）。

## 7.3 出題数を変える
`var TEST_SIZE=100;` を変更（例：50）。テストの出題上限が変わる。

## 7.4 間隔反復の間隔を変える
`var SRS_INTERVALS=[1,3,7,16];` を編集（日数の配列）。卒業条件は「box が配列長を超えたら」なので、配列長で復習回数も変わる。

## 7.5 進化段階・必要XPを変える
`STAGES` 配列の `xp`（閾値）・`name`・`desc` を編集。段階追加も可能（SVG分岐 `petSVG` の `scale`/装飾も合わせて調整）。

## 7.6 配色・ブランドを変える
`:root` のCSS変数（`--shell`,`--coral-1/2/3`,`--carrot`,`--grass`,`--gold` ほか）を変更。ヘッダーの `.kicker/.brandbar h1/.tagline`、フッターのキャッチコピーも編集可。

## 7.7 参考リンクを足す・差し替える
`REF_RULES` に `{re:/キーワード/, refs:[R("表示名","URL","official|related|academic"), ...]}` を追加。カテゴリ既定は `CAT_FALLBACK` を編集。URLは `WK+"語"`、`TBH+"ページ"`、`gs("サイト","検索語")`、`sch("検索語")` が利用可能。

## 7.8 報酬バランスを調整
`answerLearn`（XP/通貨）、`feed`（回復量）、`answerChallenge`/`answerTest`（XP）、`neckForScore`（首飾り閾値）、`tick`（減衰）の数値を編集。

## 7.9 AI生成の既定モデルを変える
`apiCfg()` の既定（`"gpt-4.1-mini"`）や、`generateAI` 内のAnthropic既定（`"claude-3-5-haiku-latest"`）を必要に応じ変更。キーは利用者がアプリ内で設定。

## 7.10 効果音・アニメを止める
効果音は「ようす」のサウンドトグルでOFF可能（実装は `beep` が `S.sound` を見る）。アニメは利用者OSの reduce-motion で自動抑制。

## 7.11 注意（編集時の鉄則）
- 1文字でも構文を壊すと全機能が無反応になり得る。**末尾 `</script></body></html>` まで欠落させない**こと。
- 文字列内の引用符・バッククォートのエスケープに注意。
- 編集後は必ず「起動・各ボタン・テスト一巡」を再確認（文書5 R-01〜R-03）。

---

🐰 *うさうさ研修工房 ／「面白きこともなき世を面白く」 ／ 本書は v3.2.0 実装に基づく事実記述。*
