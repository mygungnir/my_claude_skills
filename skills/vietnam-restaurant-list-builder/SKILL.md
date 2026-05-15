---
name: vietnam-restaurant-list-builder
description: ハノイの中〜高級レストラン100〜150店舗を網羅した検索可能なJSONリスト（hanoi-restaurants.json）を作成・更新するスキル。ユーザーが「ハノイのレストランリスト作って」「リスト更新して」「新しい店を追加して」「ミシュラン店を全部追加して」と言ったときに発動する。複数の権威ソース（Michelin Guide・OpenTable・PasGo・VELTRA・在住者向けメディアなど）から店舗を集約し、各店舗をエンリッチして検索可能なリストを構築する。**個別の店の予約手段を調べたい場合は vietnam-restaurant-booking スキルを使う**こと。**お店を提案・推薦してほしい場合は vietnam-restaurant-finder スキルを使う**こと。
---

# Vietnam Restaurant List Builder（ハノイ中〜高級店リスト構築）

ハノイの中〜高級レストランを網羅した検索可能なJSONリストを作成・更新するためのスキル。

## このスキルの目的

毎日使う `vietnam-restaurant-finder` が参照する**マスターデータ `hanoi-restaurants.json`** を作る・更新する。100〜150店舗を網羅し、用途・エリア・予算・予約手段で検索できる構造にする。

**重要な性質：**
- このSKILLは**低頻度（数ヶ月に1回）**実行される重い処理
- 1回の実行で100店舗を一気に集めるのは現実的でない
- **増分更新ができる構造**にすること（ソースごとに分けて実行可能、既存JSONに追加できる）

## 対象店舗の定義

以下に該当するレストランを集める：

| 軸 | 条件 |
|---|---|
| **予算** | 1人あたり **80万〜200万ドン**（約5千〜1.2万円）程度 |
| **シーン** | 接待・会食 / 妻・パートナーとの記念日・デート / ホテル・夜景重視 |
| **エリア** | ハノイ全域（Ba Đình / Hoàn Kiếm / Tây Hồ / Cầu Giấy / Hai Bà Trưng / Đống Đa 等） |
| **対象外** | 屋台、ローカル食堂、ファストフード、80万ドン未満の店、ファミレス系チェーン |

## ワークフロー（4フェーズ）

### Phase 1: ソース集約（Source Aggregation）

以下7ソースから店舗を集める。**ソースごとに別々に実行可能**にすること（一気に全部やらなくてもよい）。

#### ソース一覧

| 優先 | ソース | 取得方法 | 期待件数 |
|---|---|---|---|
| 1 | **Michelin Guide Hanoi** | `guide.michelin.com/vn/en/ha-noi/...` をweb_fetch。**403 がほぼ常態**なので、失敗時は即フォールバック（Wikipedia「List of Michelin-starred restaurants in Vietnam」+ 旅行メディア記事を web_search → web_fetch で**三角測量**する。詳細は `references/sources.md` § 1 を参照） | 30〜50 |
| 2 | **OpenTable Hanoi** | `opentable.com/metro/vietnam` を辿る | 30〜50 |
| 3 | **PasGo 高級カテゴリ** | `pasgo.vn/ha-noi/nha-hang/cao-cap` 等を確認 | 50〜100 |
| 4 | **TimeOut Hanoi - Best Restaurants** | web_search で記事を探してweb_fetch | 20〜40 |
| 5 | **VELTRA / KKday の代行予約対象店** | 既出ページから店名抽出 | 10〜20 |
| 6 | **在住者向け日本語メディア** | ベトナムリアルガイド、Poste-vn、ベトナムニュース等 | 20〜40 |
| 7 | **5つ星ホテル併設レストラン** | Metropole, Lotte, Sofitel, Hilton, JW Marriott 等のレストランページ | 20〜30 |

ソース別の詳しい収集方法は `references/sources.md` を参照。

#### Phase 1のアウトプット

各ソースから取得した店舗を**raw形式**で `data/raw-<source>.json` に保存：

```json
{
  "source": "michelin",
  "collected_at": "2026-05-15",
  "restaurants": [
    {
      "name_official": "Tầm Vị",
      "source_url": "https://guide.michelin.com/...",
      "source_category": "1-star",
      "raw_address": "...",
      "raw_cuisine": "Vietnamese",
      "raw_notes": "ミシュラン1つ星、伝統的なベトナム家庭料理"
    },
    ...
  ]
}
```

ソースのデータ構造はバラバラでOK（次のPhaseで正規化する）。

### Phase 2: 各店舗のエンリッチ（Enrichment）

raw データの各店舗について、`places_search` と必要に応じて `web_search` / `web_fetch` で情報を補完する。

**前提：Places API が必須**

`google_maps_place_id` / `rating` / `rating_count` / `latitude` / `longitude` のような Google 由来フィールドは、`places_search`（MCP の Google Places 連携）が**ないと埋められない**。連携が無いまま実行すると全件 `null` のリストになり、`vietnam-restaurant-finder` の地理検索が機能しないので、Phase 2 開始前に以下を確認すること：

- `places_search` 系のツールがロード済か（無ければユーザーに **MCP Places 連携を入れてから再実行**を依頼する）
- 連携がどうしても用意できない場合：lat/lng は番地から街区レベルの近似値を入れ、`google_maps_place_id` は `null`、`notes` に `"NEEDS_PLACES_API_LOOKUP"` フラグを残す（次回再実行時に検出できるように）

#### 取得する情報

```json
{
  "id": "tam-vi",                       // ファイルパスやURLに使えるslug
  "name_official": "Tầm Vị",
  "name_alt": ["タムヴィー"],            // 日本語表記の通称があれば
  "address": "4A Yết Kiêu, Hoàn Kiếm, Hà Nội",
  "district": "Hoàn Kiếm",              // 区
  "latitude": 21.0234,
  "longitude": 105.8456,
  "phone": "+84 943 143 686",
  "phone_zalo_compatible": true,         // 携帯番号かどうか（09,03,05,07,08始まり）
  "website": "https://...",
  "google_maps_place_id": "ChIJ...",
  "google_maps_url": "https://maps.google.com/?cid=...",
  "rating": 4.6,
  "rating_count": 423,
  "price_level_google": 3,               // Google price_level (1-4)
  
  "cuisine": ["Vietnamese", "Modern Vietnamese"],   // 料理ジャンル（複数可）
  "price_range_vnd_per_person": [800000, 1500000],  // 円ではなくドン
  
  "scenes": ["business", "date", "japanese-guests"], // 用途タグ（後述）
  "ambience": ["fine-dining", "traditional"],        // 雰囲気タグ
  
  "michelin_status": "1-star",           // "1-star" | "2-star" | "3-star" | "bib-gourmand" | "selected" | null
  "is_hotel_restaurant": true,           // ホテル併設か
  "hotel_name": "Sofitel Metropole",     // ホテル名（あれば）
  "has_view": true,                       // 景観・夜景重視店か
  
  "booking_channels": {
    "phone": "+84 943 143 686",
    "official_website_booking": "https://booking.example.com/...",  // 公式予約ページがあれば
    "pasgo_url": "https://pasgo.vn/nha-hang/...",
    "opentable_url": null,
    "tablecheck_url": null,
    "capichi_url": null,
    "facebook_url": "https://facebook.com/...",
    "facebook_messenger": "https://m.me/...",
    "instagram_url": "https://instagram.com/...",
    "zalo_url": "https://zalo.me/84xxx"   // 携帯番号の場合のみ
  },
  
  "sources": ["michelin", "veltra"],     // 元のソース（複数可、重複時に統合）
  "japanese_support": false,             // 日本語スタッフ有無
  "notes": "ミシュラン1つ星、伝統的なベトナム家庭料理が楽しめる",
  "last_updated": "2026-05-15"
}
```

#### タグ語彙（重要：これに従う）

**scenes（用途）：**
- `business` ... 接待・会食向け
- `casual-colleagues` ... 同僚とのカジュアル会食
- `japanese-guests` ... 日本からの来客を連れて行ける
- `date` ... 妻・パートナーとの記念日・デート
- `family` ... 家族・子連れ
- `large-group` ... 大人数（10名以上）可

**ambience（雰囲気）：**
- `fine-dining` ... 高級・フォーマル
- `traditional` ... 伝統的・古民家風
- `modern` ... モダン・スタイリッシュ
- `rooftop` ... ルーフトップ
- `view` ... 景観あり（湖・川・夜景）
- `garden` ... ガーデン・テラス席
- `private-room` ... 個室あり

**cuisine（料理）：**
- `Vietnamese` `Modern Vietnamese` `French` `Italian` `Japanese` `Chinese` `Korean` `Thai` `Indian` `Mediterranean` `Steakhouse` `Seafood` `Fusion` `Vegetarian`

`booking_channels` の調査ロジックは `vietnam-restaurant-booking` SKILL の手順を流用してよい。重複処理を書かないこと。

#### Phase 2のアウトプット

`data/enriched-<source>.json` に保存。raw と同じファイル名にしないこと（後で見比べるため）。

### Phase 3: マージ・重複排除（Merge & Dedupe）

複数のソースから取得した店舗を1つの JSON に統合する。

#### 重複判定ロジック

以下の順で判定：

1. **`google_maps_place_id` が一致** → 同一店舗（最も確実）
2. **緯度経度が 50m 以内 + 店名が部分一致** → 同一店舗
3. **電話番号が一致** → 同一店舗
4. **店名がベトナム語的に完全一致 + 住所の通り名が一致** → 同一店舗の可能性高い（手動確認推奨）

#### 重複時の情報統合

- `sources` には全ソース名を配列で残す（例：`["michelin", "veltra", "pasgo"]`）
- `notes` は最も詳しいものを採用、他のソースの記述は追記
- `booking_channels` はソースごとに違うURLが取れることがあるので、全部マージ
- `michelin_status` は Michelin Guide ソースの値を優先
- `name_alt` には日本語表記やローカル通称をすべて入れる

#### 注意事項

- **同名チェーン店の別店舗は別エントリ**として保持する（例：Pizza 4P's Ben Thanh と Pizza 4P's Le Thanh Ton）
- 「**偽住所店舗**」を検知する（前のテストで判明した既知の問題）：
  - Chả Cá Thăng Long のように、公式サイトが偽住所を警告している店がある
  - 公式サイトをweb_fetchして警告文言の有無を確認
  - `notes` に「偽店舗注意：公式は X 住所」と明記

### Phase 4: タグ付け・カテゴリ分類

エンリッチで取れない（取りこぼした）タグを補完する。LLM判断が必要なフィールド：

- `scenes` の `business` / `date` / `japanese-guests` 判定
- `ambience` の判定
- `has_view` の判定
- `japanese_support` の判定

これらは各店の `notes` や口コミから読み取って付ける。タグ付けの基準は `references/tagging-guide.md` を参照。

## 最終アウトプット

`data/hanoi-restaurants.json` （これが `vietnam-restaurant-finder` が参照するマスターファイル）

```json
{
  "metadata": {
    "last_updated": "2026-05-15",
    "total_count": 127,
    "sources_used": ["michelin", "opentable", "pasgo", "veltra", "kkday", "timeout", "veltra-jp-media", "hotels"],
    "version": "1.0"
  },
  "restaurants": [
    { /* 上記スキーマの店舗1 */ },
    { /* 上記スキーマの店舗2 */ },
    ...
  ]
}
```

## 実行戦略：一気にやらず、段階的に

100〜150店舗を一度に集めるのは現実的でないので、**ユーザーには段階的な実行を提案**する。

### 推奨実行プラン

ユーザーから「リスト作って」と言われたら、初回はこう聞き返す：

> ハノイの中〜高級店リストを作るには 100〜150 店舗の調査が必要で、一気にやると時間がかかります。以下の優先度で順番にやりますがどうでしょう？
> 
> **第1弾（最重要・約30〜50店舗、1セッション）：**
> - Michelin Guide Hanoi 全店
> - OpenTable Hanoi 全店
> - 5つ星ホテル併設レストラン
>
> **第2弾（追加・約30〜50店舗）：**
> - PasGo の高級カテゴリ
> - VELTRA / KKday の代行予約対象店
>
> **第3弾（さらに追加・約20〜40店舗）：**
> - TimeOut Hanoi
> - 在住者向け日本語メディアからのキュレーション
>
> どこまで進めますか？

各弾ごとに JSON を更新し、いつでも中断・再開できる構造にする。

### 中断・再開のしくみ

各 Phase の中間生成物（`raw-<source>.json`、`enriched-<source>.json`）を残しておく。最終的な `hanoi-restaurants.json` はマージ結果。再実行時は：

1. 既存の `hanoi-restaurants.json` を読む
2. 「まだ取り込んでいないソース」だけ Phase 1〜2 を実行
3. 既存マスターに対して Phase 3（マージ）

## 詳細ドキュメント

実装の詳しい手順は以下を参照：
- `references/sources.md` — 7ソースそれぞれの収集手順とURL例
- `references/tagging-guide.md` — scenes / ambience の判定基準
- `references/schema.md` — 店舗JSONの完全スキーマ定義

## 既知の落とし穴

1. **places_search で電話番号が取れない店がある**：公式サイト / Facebook ページ / 他のレビューサイトを当たれば見つかることが多い
2. **同じ店の住所表記がソースによって違う**：通り名（例：`P. Linh Lang` vs `Linh Lang street`）の差は無視する
3. **偽店舗（住所改ざん）に注意**：公式サイトの警告文をチェック
4. **チェーン店の支店判別**：必ず支店ごとに別エントリにする
5. **ベトナム語の声調記号**：正式名は声調付きで保存。検索時は声調なし版も用意（`name_normalized`）
6. **Michelin 公式 (`guide.michelin.com`) は WebFetch がほぼ常に 403**：bot ブロックされる。1 回だけ試して失敗したら即 Wikipedia + 旅行メディアの**三角測量**に切り替える。詳細フローは `references/sources.md` § 1 の「公式が 403 だったときのフォールバック」を参照。
7. **Places API（`places_search`）が無いと `google_maps_place_id` / `rating` / `rating_count` / `latitude` / `longitude` が埋まらない**：Phase 2 開始前に必ずツールの有無を確認する。無い状態で実行するとマスター JSON は構造的に正しくても**地理検索が機能しない**ので、`vietnam-restaurant-finder` 側で「Places lookup 未完了の店」を検出する必要が出る。null の場合は `notes` に `"NEEDS_PLACES_API_LOOKUP"` フラグを残し、次回再実行時にバッチで埋め直す。

## このSKILLの境界（重要）

- 個別店の予約手段だけを調べたい → `vietnam-restaurant-booking` を使う
- 店を提案してほしい → `vietnam-restaurant-finder` を使う
- このSKILLは**リストを作るだけ**。検索やリコメンドはしない。
