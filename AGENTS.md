# AGENTS.md

月次のお金の配分計画アプリ「お金計画」の開発メモ。コードを変更する前に読むこと。

## 方針

- `index.html` 1 ファイルに HTML・CSS・JS をすべて同梱する（アイコン画像と manifest だけは別ファイル）。ビルド不要。外部ライブラリ・CDN・Web フォントは使わない
- サーバー、スプレッドシート連携、外部 API への通信はしない（現時点）
- データはブラウザの `localStorage` にだけ保存する
- 公開リポジトリなので、**実際の金額・個人の項目名をコードやドキュメントに入れない**。初期データはカテゴリ名だけ
- スマホ優先。`input` / `select` / `textarea` は **16px 以上**（iOS Safari の自動拡大を防ぐ）。横スクロールを出さない（幅 320px でも崩れないこと）
- 色は `:root` の CSS 変数で定義し、ダークモード（`prefers-color-scheme` と `data-theme`）でも読めるようにする
- GitHub へ push する前に、必ずオーナーに確認を取る

## ファイル構成

| ファイル | 内容 |
|---|---|
| `index.html` | アプリ本体 |
| `icon.svg` | アイコンの元データ（ファビコンにも使う）。配分のドーナツ（固定費・変動費・貯金・予備費の色）と ¥ |
| `apple-touch-icon.png` / `icon-192.png` / `icon-512.png` | `icon.svg` から書き出した PNG（iOS のホーム画面用 180px、Android 用 192px・512px） |
| `manifest.webmanifest` | ホーム画面に追加したときの名前・アイコン・表示方法 |
| `README.md` | 利用者向けの説明、パスコードのハッシュの作り方、GitHub Pages の公開手順 |
| `AGENTS.md` | このファイル（構成と計算ルール） |
| `CLAUDE.md` | `@AGENTS.md` の 1 行だけ |

## `index.html` の中の構成（`<script>` 内の順）

1. **設定** … `PASSCODE_HASH`、`SCHEMA_VERSION`、`DEFAULT_CATEGORIES`、`ITEM_FIELDS`
2. **ストレージ層（`Storage`）** … `localStorage` に触れてよいのはここだけ
   - `load()` / `save(data)` / `clear()` … データ全体（1 つの JSON）の読み書き。**Promise を返す**
   - `isUnlocked(hash)` / `setUnlocked(hash)` / `clearUnlock()` … 端末ごとのロック解除フラグ（同期しない）
   - 将来 GAS（Google スプレッドシート）に切り替えるときは、この `Storage` の中身だけを差し替える
3. **データモデル** … `createInitialData()`、`newItem()`、`newPlan()`、`copyPlan()`、`toAmount()`、`migrate()`、`normalizeData()`
4. **計算（純粋関数）** … `daysInMonth()`、`calcPlan()`、`calcVariableItem()`
5. **画面** … ホーム（`renderHome`）、入力（`renderInputPane` ほか）、設定（`renderSettings` ほか）
   - 画面の切り替えは URL のハッシュ: `#home` / `#input` / `#input/<income|fixed|savings|variable>` / `#settings`
   - 設定のカテゴリは、左のつまみ（≡）を押さえてドラッグで並べ替える（`setupCategoryDrag()`、Pointer Events で指・マウス共通。画面端で自動スクロール。つまみにフォーカスして上下キーでも動かせる）
   - ホームは「＋ 入るお金」「− 出ていくお金・よけておくお金」「％ 割合」の 3 グループ。各行をタップすると `#input/<種類>` でその入力タブを開く
6. **起動** … ロック確認 → `startApp()`

画面のコードは `localStorage` を直接触らないこと。変更したら `commit()` を呼ぶ（`state.data.updatedAt` を更新し、少し待ってから `Storage.save()`）。項目を変更したら `touchItem(item)` で項目とプランの `updatedAt` も更新する。

## データ形式（schemaVersion 3）

データ全体を 1 つの JSON として扱う。書き出し／読み込みもこの形のまま。

```jsonc
{
  "schemaVersion": 3,
  "updatedAt": "2026-09-29T00:00:00.000Z",         // データ全体の最終更新（ISO 8601）
  "categoriesUpdatedAt": "2026-09-29T00:00:00.000Z",
  "categories": {
    "income":      ["給与", "副業", "賞与", "臨時収入", "その他収入"],
    "variable":    ["日用品", "娯楽", "未分類", "外食", "交際費", "軽食", "散髪", "通院", "食費", "釣り", "固定費", "その他"],
    "savingsType": ["現金", "投資", "持ち株会"]
  },
  "plans": {
    "2026-09": {                                    // キーは対象月（yyyy-MM）
      "month": "2026-09",
      "createdAt": "…", "updatedAt": "…",
      "variableMode": "items",                       // 変動費の入れ方: "items"（項目別）/ "total"（全体で1つの金額）
      "variableTotal": 0,                            // "total" のときの変動費予算
      "income":   [{ "id": "uuid", "enabled": true, "category": "給与", "amount": 0, "memo": "", "updatedAt": "…" }],
      "fixed":    [{ "id": "uuid", "enabled": true, "name": "家賃", "amount": 0, "payDay": "27", "memo": "", "updatedAt": "…" }],
      "savings":  [{ "id": "uuid", "enabled": true, "type": "投資", "name": "NISA", "amount": 0, "memo": "", "updatedAt": "…" }],
      "variable": [{ "id": "uuid", "enabled": true, "category": "食費", "amount": 0, "memo": "", "updatedAt": "…" }]
    }
  }
}
```

- 金額 `amount` は **0 以上の整数（円）の数値**。入力は全角数字・カンマ・¥ を許容し、`toAmount()` で数値化する
- 対象月は `yyyy-MM` 文字列
- `variableMode` が `"total"` の月は、変動費予算合計 ＝ `variableTotal`。項目別の `variable` 配列は消さずに残し、`"items"` に戻すとその合計で計算する。全体入力に切り替えたとき `variableTotal` が 0 なら、有効な項目の合計を初期値にする
- 月のコピーでは `variableMode` と `variableTotal` も引き継ぐ
- 支払日 `payDay` は文字列: `""`（未設定）/ `"1"`〜`"31"` / `"末"`（月末）
- 日時は ISO 8601 文字列（UTC）
- 各項目は一意の `id`（`crypto.randomUUID()`）と `updatedAt` を持つ。将来の端末間同期で突き合わせに使う
- 月をコピーしたときは、項目の `id` と `updatedAt` を振り直す
- カテゴリは名前の配列。項目はカテゴリを **名前で** 持つ。カテゴリを削除しても既存の項目はそのまま残り、選択肢に「（未登録）」付きで表示される
- 項目の削除は今は物理削除。同期を入れるときは、削除の伝搬のために墓標（`deletedAt` など）を検討する
- 形式を変えるときは `SCHEMA_VERSION` を上げ、`migrate()` に旧形式からの変換を足す。`normalizeData()` は読み込み時・インポート時に必ず通す。古い形式を読み込んだら、起動時にすぐ新しい形式で保存し直す
- 変更履歴:
  - v1 → v2: `variableMode` / `variableTotal` を追加（v1 の月は `"items"`、`0` にする）
  - v2 → v3: 変動費カテゴリを現在の初期値（オーナーの家計簿アプリと同じ項目・順番）に置き換え。自分で追加したカテゴリは後ろに残す。項目の「外食費」は「外食」に改名。なくなったカテゴリの項目は、金額もメモも空なら削除し、どちらかがあれば残す
- 変動費の項目は、入力画面ではカテゴリマスタの順に並べて表示する（`sortByCategoryOrder()`。保存データの順番は変えない）

## 計算ルール

**有効（`enabled: true`）な項目だけ** を集計する。

| 名前 | 計算 |
|---|---|
| 収入合計 | 有効な収入の `amount` の合計 |
| 固定費合計 | 有効な固定費の合計 |
| 貯金合計 | 有効な貯金・投資の合計。種別ごとの小計も出す（マスタの順 → マスタにない種別 → 種別なし） |
| 変動費予算合計 | `variableMode` が `"total"` なら `variableTotal`、`"items"` なら有効な変動費の合計 |
| 支出＋貯金 | 固定費合計 ＋ 変動費予算合計 ＋ 貯金合計 |
| 予備費 | 収入合計 −（支出＋貯金） |
| 貯蓄率 | 貯金合計 ÷ 収入合計（収入 0 なら「—」） |
| 収入比の配分 | 固定費・変動費・貯金投資・予備費それぞれ ÷ 収入合計 |

変動費の各項目（自動計算）:

| 名前 | 計算 |
|---|---|
| 週あたり目安 | 月予算 × 12 ÷ 365 × 7 |
| 1日あたり目安 | 月予算 ÷ 対象月の日数 |
| 変動費内割合 | 月予算 ÷ 変動費予算合計（無効な項目や合計 0 のときは「—」） |

変動費予算合計の週あたり・1日あたりも同じ式で出す（全体で入力したときも表示する）。

表示は円単位で四捨五入、割合は小数 1 桁。

### 予備費がマイナスのとき

- ホームの予備費を赤字にし、「予算オーバー：¥X 足りません」の警告と入力画面へのボタンを出す
- 入力画面上部のミニサマリーを赤くする
- 配分バーは「支出＋貯金」を全幅として描き、収入の位置に線を引いて、超えた部分を赤の斜線で示す

## アイコン

`icon.svg` を変えたら、PNG 3 つも同じ見た目で書き出し直す（ヘッドレスブラウザで SVG を 180 / 192 / 512px に描いてスクリーンショットを撮るなど）。背景は角まで塗る（iOS が角を丸める）。Android の maskable 用に、絵柄は中心から半径 40% の円の内側に収める。

## パスコード

- `PASSCODE_HASH` にはパスコードの SHA-256（16 進・小文字）だけを書く。パスコードそのものは書かない
- 入力値を Web Crypto API（`crypto.subtle.digest`）でハッシュ化して比較する。`https://` か `localhost` で開く必要がある
- 正しければ、そのハッシュを `localStorage` に記録する。記録値が現在の `PASSCODE_HASH` と一致する間はロック画面を出さない（ハッシュを変えると全端末で再入力になる）
- 設定画面の「この端末のロック解除を取り消す」で記録を消す（データは消えない）
- ハッシュの作り方は README.md を参照

## 動作確認のしかた

```sh
python3 -m http.server 8000   # http://localhost:8000/
```

幅 390px と 320px、ライト／ダークで次を確認する: ロック画面（誤りと正解）、各タブでの追加・編集・削除・スイッチ、予備費マイナス時の表示、再読み込み後もデータが残ること、月の新規作成（コピー／空）、カテゴリの追加・並べ替え・削除、JSON の書き出し・読み込み、ロック解除の取り消し。
