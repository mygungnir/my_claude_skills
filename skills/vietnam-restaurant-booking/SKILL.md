---
name: vietnam-restaurant-booking
description: ハノイの特定レストランの予約手段を1店舗ぶん集めて返すスキル。電話・Zalo・Facebook Messenger・PasGo URL・OpenTable URL・公式予約フォーム・VELTRA 代行予約の有無を、`hanoi-restaurants.json` 優先で取得し、足りなければ web で補完する。ユーザーが「Tầm Vị の予約方法を教えて」「Hibana の電話番号は？」「ここってZalo で予約できる？」「VELTRA で代行できる？」「ハノイの〇〇の公式予約ページある？」と言ったときに発動する。**店を提案・推薦してほしい場合は `vietnam-restaurant-finder` を使う**こと。**マスターリストを作る・更新する場合は `vietnam-restaurant-list-builder` を使う**こと。
---

# Vietnam Restaurant Booking（ハノイ単一店の予約手段リターン）

ハノイの特定 1 店舗について「**どうやって予約するか**」を最短で返す Capability スキル。

## このスキルの目的

`vietnam-restaurant-finder` が「店を提案する」のに対し、こちらは「**選ばれた 1 店をどう予約するか**」だけを返す。

- マスター JSON `../vietnam-restaurant-list-builder/data/hanoi-restaurants.json` の `booking_channels` を最優先
- 足りなければ web で取りに行く（公式サイト → Facebook → Google Maps）
- 日本人ユーザーが「ベトナム語で電話するのが不安」な前提で、**Zalo / Facebook Messenger / VELTRA 代行予約**などの非音声チャネルを優先して案内する

## 発動条件

以下のような会話で必ず使う：

- 「〇〇（店名）の予約方法を教えて」「電話番号は？」「Zalo ある？」
- 「ここって何で予約できる？」「公式予約ページある？」
- 「VELTRA 代行で予約できる？」「日本語通じる窓口は？」
- `vietnam-restaurant-finder` の出力後に「これ予約したい」と言われたとき

「**スキル**」という単語が無くても、特定 1 店の**予約導線を出す**文脈なら自動的に発動する。

## 入力

- **店名 or id**（必須）— 例: `"Tầm Vị"`, `"tam-vi"`, `"ヒバナ"`, "Hibana by Koki"
- ユーザーの言語適性（任意）— 例: 「ベトナム語できない」「英語なら OK」「日本語のみ」

店名が曖昧（"ハノイの和食屋" 等）なら finder へ差し戻し。**1 店に特定できるまで先へ進まない**。

## ワークフロー

### Step 1: マスター JSON でルックアップ

`../vietnam-restaurant-list-builder/data/hanoi-restaurants.json` を読み、以下の順で同定：

1. `id` 完全一致（例: `"tam-vi"`）
2. `name_official` 完全一致（声調記号あり / 無し両方試す → `name_normalized` で照合）
3. `name_alt` のいずれかと一致（日本語通称 "タムヴィー" 等）
4. 部分一致（例: ユーザーが "Hibana" → `hibana-by-koki` にヒット）— 候補が複数出たら確認質問

同定できたら **Step 2** へ。マスターに無ければ **Step 4**（web 補完）。

### Step 2: booking_channels をベースに整形

`r.booking_channels` を読み、以下の構造で出力する。**null フィールドは省略しない**で「未取得」と明示する（情報の有無を区別できるように）。

| チャネル | 値 | 備考 |
|---|---|---|
| 電話 | `r.phone` | 国際表記 `+84 ...` |
| Zalo | `booking_channels.zalo_url` | 携帯番号の場合のみ存在 |
| 公式予約 | `booking_channels.official_website_booking` | あれば最優先で案内 |
| PasGo | `booking_channels.pasgo_url` | ベトナム語だが日本人も使いやすい UI |
| OpenTable | `booking_channels.opentable_url` | 英語、海外クレカ対応 |
| TableCheck | `booking_channels.tablecheck_url` | 日本人ユーザーに最も馴染み |
| Capichi | `booking_channels.capichi_url` | 日本食メイン |
| Facebook | `booking_channels.facebook_url` | 営業時間・空席アナウンス確認用 |
| Messenger | `booking_channels.facebook_messenger` | チャットで予約できる店多い |
| Instagram | `booking_channels.instagram_url` | DM 予約対応店も |
| VELTRA 代行 | `sources` に "veltra-jp-media" 含めば「あり」 | 日本語で代行予約可能 |

### Step 3: Zalo URL の自動構築

`phone` があり `zalo_url` が null の場合、`phone_zalo_compatible` を見て**自力で構築**する：

#### 携帯番号判定（Zalo 利用可能）

`+84` 直後の数字が以下なら携帯番号 = Zalo 利用可：
- `9x` (古典的 Viettel/MobiFone/Vinaphone 番号)
- `3x` (Viettel 拡張)
- `5x` (Vietnamobile / Reddi)
- `7x` (Mobifone 拡張)
- `8x` (Vinaphone 拡張)

Hanoi 固定電話 `+84 24` は Zalo 不可。

#### 構築ルール

`+84 ` の後の空白を除いて `https://zalo.me/84xxxxxxxxx` 形式で組む：

| 元の `phone` | 結果 `zalo_url` |
|---|---|
| `+84 943 143 686` | `https://zalo.me/84943143686` |
| `+84 96 632 3131` | `https://zalo.me/84966323131` |
| `+84 24 3826 6919` | （null — 固定電話のため不可） |

### Step 4: マスターに無い / 情報が薄いときの web 補完

`booking_channels` がほぼ null（電話以外の全チャネルが null）、またはマスターに該当店が無い場合：

#### 補完手順

1. **`web_search`** で `"<店名> Hanoi reservation"` `"<店名> Facebook"` `"<店名> 公式サイト"` 等を検索
2. **公式サイト** が見つかれば `web_fetch` で予約ページ / 電話 / Facebook リンクを抽出
3. **Facebook ページ** が見つかれば URL を取得（Messenger 短縮 URL は `m.me/<page>`）
4. **`places_search`**（MCP Places 連携が使える場合のみ） で電話・Place ID を補完
5. 取得した情報は**ユーザー出力に使うと同時に、Step 4.5 でマスター JSON にも書き戻す**（次回 builder 実行を待たない）

#### 偽店舗の警告チェック

`vietnam-restaurant-list-builder` SKILL の既知の落とし穴 #3 に従って、公式サイトに「偽店舗注意」「giả mạo」「fake address」等の警告文がないか必ず確認する。あれば**最優先で出力に載せる**：

> ⚠️ この店は公式サイトで「偽の住所を名乗る別店舗があります」と警告しています。`<公式が示す正しい住所>` のみが本物です。

**偽店舗警告が見つかった場合は Step 4.5 の書き戻しを保留する** — 住所が公式と食い違っている可能性があるので、書き戻し前に `vietnam-restaurant-list-builder` での再検証を促す。

### Step 4.5: マスター JSON への補完書き戻し（in-place update）

web 補完で得た情報を **`../vietnam-restaurant-list-builder/data/hanoi-restaurants.json` に in-place で追記**する。次回 builder 実行を待たない。

#### 対象フィールド

書き戻し**可**：
- `booking_channels.official_website_booking`
- `booking_channels.pasgo_url`
- `booking_channels.opentable_url`
- `booking_channels.tablecheck_url`
- `booking_channels.capichi_url`
- `booking_channels.facebook_url`
- `booking_channels.facebook_messenger`（`m.me/<page>` 形式）
- `booking_channels.instagram_url`
- `booking_channels.zalo_url`（Step 3 の自動構築結果）
- `website`（公式サイト URL）

書き戻し**不可**（builder の責務として残す）：
- `phone` — 既存値があってウェブ側と差異がある場合、別店舗を誤同定している可能性が高いので触らない
- `address`, `latitude`, `longitude`, `district` — 偽店舗の混入を避けるため変更しない
- `price_range_vnd_per_person`, `cuisine`, `scenes`, `ambience` — 価値判断を伴うので builder の責務
- `michelin_status`, `is_hotel_restaurant`, `hotel_name` — 同上
- `google_maps_place_id`, `rating`, `rating_count` — Places API での確定値のみ書く（手動推測しない）

#### 書き戻し条件

**「既存値が null のときだけ上書きする」**。non-null の値は絶対に踏まない（builder が確定させた正準値を尊重）。

例：
```
master.booking_channels.facebook_url = null → web 経由で見つけた値で埋める ✅
master.booking_channels.facebook_url = "https://facebook.com/old" → 触らない（builder の責務） ❌
```

#### 追加更新

- `last_updated` (店舗エントリ): 当日付に更新
- `notes`: 末尾に `" / [YYYY-MM-DD] booking-skill 補完: <追加されたチャネル名のリスト>"` を append（監査トレイル）
- `sources`: 新しいソース種類を発見した場合のみ追加（例: 公式サイトのみで新情報を得た場合は変更不要）
- `metadata.last_updated`: ファイル全体の最終更新日も同時に更新

#### 実装手順（書き戻しの実行コード）

```python
import json
from datetime import date

MASTER = "../vietnam-restaurant-list-builder/data/hanoi-restaurants.json"
TODAY = date.today().isoformat()

with open(MASTER) as f:
    data = json.load(f)

# 該当エントリを id で取得
entry = next(r for r in data["restaurants"] if r["id"] == "<resolved_id>")

# 書き戻し可フィールドだけを null チェックして更新
discoveries = {
    "booking_channels.official_website_booking": "https://...",
    "booking_channels.facebook_url": "https://...",
    "website": "https://...",
}
applied = []
for path, value in discoveries.items():
    if value is None:
        continue
    parts = path.split(".")
    target = entry
    for p in parts[:-1]:
        target = target[p]
    if target.get(parts[-1]) is None:
        target[parts[-1]] = value
        applied.append(parts[-1])

if applied:
    entry["last_updated"] = TODAY
    entry["notes"] = (entry.get("notes") or "") + f" / [{TODAY}] booking-skill 補完: {', '.join(applied)}"
    data["metadata"]["last_updated"] = TODAY

# スキーマ再バリデーション（builder と同じルールを再実行）して合格を確認してから書き出す
# 不合格なら abort して書き戻さない
# ...validation here...

with open(MASTER, "w") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

#### Step 4.5 のスキップ条件（書き戻ししない場合）

以下のいずれかに該当する場合は **書き戻しを行わず、Step 5 の出力に「※マスター JSON 未反映」と明記する**：

1. **マスターに該当店が無い**（id 解決できなかった）— 新規店の追加は builder の責務
2. **偽店舗警告が見つかった** — 信頼性チェックが先
3. **検索で複数候補が出て同定が不確実**（同名別店の可能性）
4. **書き戻し後の re-validation が失敗** — schema 不整合をマスターに残さない
5. **ユーザーが明示的に「マスターは触らないで」と言った**

#### 書き戻し後のユーザー通知

Step 5 の出力の最後に、書き戻した内容を 1 行で報告：

> ✅ マスター JSON に以下を追記しました: `official_website_booking`, `facebook_url`, `instagram_url`（[hanoi-restaurants.json](../vietnam-restaurant-list-builder/data/hanoi-restaurants.json) → `the-east-indochine`）

これでユーザーが「次回 finder で出てくるときには情報が増えている」と納得できる。

### Step 5: 出力（ユーザー向け整形）

#### フォーマット例

> **Tầm Vị** (Ba Đình, Michelin 1-star) の予約導線：
>
> | チャネル | 値 | おすすめ度 |
> |---|---|---|
> | 📞 電話 | `+84 96 632 3131`（ベトナム語/英語） | ⭐ ベトナム語ができれば最速 |
> | 💬 Zalo | https://zalo.me/84966323131 | ⭐⭐⭐ **日本人ユーザー推奨**（テキストでやり取り、Google 翻訳貼り付け可） |
> | 🏨 VELTRA 代行予約 | あり（[VELTRA Hanoi](https://www.veltra.com/jp/asia/vietnam/hanoi/a/186726)） | ⭐⭐⭐ **日本語で予約完了**（3-4 日前までに依頼） |
> | 🌐 公式予約フォーム | 未取得 | — |
> | 📘 Facebook | 未取得 | — |
> | 🍽️ PasGo | 未取得 | — |
>
> **推奨ルート**：ベトナム語に不安があれば **VELTRA 代行予約 > Zalo メッセージ > 電話**の順で試すのが安全。Michelin 1-star かつ人気店のため、デート / 接待での確実な予約には 1 週間前を目安に動くのが無難。

#### 出力の優先順位ルール

1. **VELTRA 代行予約あり**（`sources` に "veltra-jp-media" 含む）→ 最上位で案内、日本語完結を強調
2. **公式予約フォーム**（ホテル系で多い）→ 次点
3. **Zalo**（携帯番号がある店）→ 日本人ユーザーが最もカジュアルに使える
4. **TableCheck / OpenTable / PasGo** → 国際的予約プラットフォーム
5. **Messenger / Instagram DM** → 個人経営店・小箱に多い
6. **電話**（音声）→ 最終手段として明記

### Step 6: フォローアップ

- 「Zalo の使い方は？」→ 「Zalo アプリインストール → リンクを開く → "Add Friend" → メッセージで予約内容を送る」を案内
- 「VELTRA で具体的にどう頼む？」→ VELTRA のサービスページを案内（`https://www.veltra.com/jp/asia/vietnam/hanoi/a/186726`）
- 「キャンセル料は？」→ マスター JSON には載っていないので「公式 / VELTRA で要確認」と返す
- 「ドレスコードは？」→ マスター JSON の `notes` に書いてあれば返す（例: Backstage は Smart casual）

## 携帯番号 → Zalo URL 変換 早見表

| 入力 phone | mobile? | zalo_url |
|---|---|---|
| `+84 9xx xxx xxx` | ✅ | `https://zalo.me/849xxxxxxxx` |
| `+84 3xx xxx xxx` | ✅ | `https://zalo.me/843xxxxxxxx` |
| `+84 5xx xxx xxx` | ✅ | `https://zalo.me/845xxxxxxxx` |
| `+84 7xx xxx xxx` | ✅ | `https://zalo.me/847xxxxxxxx` |
| `+84 8xx xxx xxx` | ✅ | `https://zalo.me/848xxxxxxxx` |
| `+84 24 xxxx xxxx` | ❌ Hanoi 固定電話 | null |
| `+84 28 xxxx xxxx` | ❌ HCMC 固定電話 | null |

## VELTRA 代行予約の使い方（よくあるフォローアップ）

VELTRA Hanoi 予約代行サービス：`https://www.veltra.com/jp/asia/vietnam/hanoi/a/186726`

- **3-4 日前まで**に予約フォームから希望日時・人数・店名を送信
- ベトナム在住の日本人スタッフが代行
- 料金：店ごとに変動（VELTRA 経由の手数料が乗る）
- **対応店一覧**は VELTRA の公式ページ参照（マスター JSON 上は `sources` に "veltra-jp-media" 含むエントリ）

## このスキルの境界（重要）

- **店を提案・推薦する** → `vietnam-restaurant-finder`
- **新規店の追加 / 多ソース横断スキャン / 価値判断系フィールドの更新（cuisine, scenes, price 等）** → `vietnam-restaurant-list-builder`
- **このスキルは 1 店ぶんの予約手段を返す + 発見した booking_channels をマスター JSON に追記する**
  - 既存値が `null` のフィールドだけを埋める（non-null は絶対に踏まない）
  - 書き換え可は `booking_channels.*`, `website` のみ — `address` / `phone` / `price_range_vnd_per_person` / `cuisine` 等は触らない
  - 詳細は Step 4.5 を参照
- 複数店の比較・推薦はしない（finder の責務）

ユーザーが「予約しといて」と言っても**実際の予約代行はしない**（VELTRA や店の電話を案内するに留まる）。代理予約はユーザー本人 / VELTRA / ホテルコンシェルジュの仕事。

## 既知の落とし穴

1. **`booking_channels` がほぼ全 null の店がある** — wave 2 で追加した PasGo / VELTRA 経由店は phone が null。public な web fetch で補完が必要
2. **偽店舗（fake address）** — 公式サイトの警告文を必ず確認（builder の既知の落とし穴 #3）
3. **Facebook ページが個人 owner の本名でついている小箱** — 検索しても出てこない場合 Instagram のほうがヒットしやすい
4. **VELTRA 対応店リストの更新ラグ** — VELTRA 側で対応店が増減することがある。マスター JSON の "veltra-jp-media" タグは収集時点のスナップショット。確実な予約可否は VELTRA に最終確認するよう促す
5. **Capella Hanoi / Sofitel Metropole などホテル併設店** — ホテルのコンシェルジュ電話 1 本で全店一括予約できることが多い。複数店ハシゴ希望なら**ホテル代表番号を案内**するのが効率的

## 詳細ドキュメント

- `references/schema.md` — `booking_channels` 完全スキーマ（builder と共有）
- `../vietnam-restaurant-list-builder/data/hanoi-restaurants.json` — 検索対象本体
- `../vietnam-restaurant-list-builder/data/raw-veltra-jp-media.json` — VELTRA 対応店のローカル参照（マスターから外れた店もここに残っている）
