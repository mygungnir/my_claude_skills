# Schema（店舗JSONスキーマ定義）

各店舗エントリの完全な JSON スキーマ定義。`vietnam-restaurant-finder` がこのスキーマに依存して検索するので、フィールド名は厳守。

---

## マスターファイル全体構造

```json
{
  "metadata": {
    "last_updated": "2026-05-15",
    "total_count": 127,
    "sources_used": ["michelin", "opentable", "pasgo", "veltra", "kkday", "timeout", "veltra-jp-media", "hotels"],
    "version": "1.0"
  },
  "restaurants": [
    { /* RestaurantEntry スキーマ */ }
  ]
}
```

---

## RestaurantEntry スキーマ

### 識別・基本情報

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `id` | string | ✅ | slug形式の一意ID。例: `"tam-vi"`, `"pizza-4ps-ben-thanh"` |
| `name_official` | string | ✅ | 正式店名（ベトナム語、声調記号付き）。例: `"Tầm Vị"` |
| `name_normalized` | string | ✅ | 検索用の声調なし版。例: `"tam vi"` |
| `name_alt` | string[] | | 別名・通称（日本語表記など）。例: `["タムヴィー"]` |

### 位置情報

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `address` | string | ✅ | 完全な住所 |
| `district` | string | ✅ | 区。`"Hoàn Kiếm"`, `"Ba Đình"`, `"Tây Hồ"`, `"Cầu Giấy"`, `"Hai Bà Trưng"`, `"Đống Đa"`, `"Long Biên"`, `"Nam Từ Liêm"`, `"Bắc Từ Liêm"` のいずれか |
| `latitude` | number | ✅ | 緯度 |
| `longitude` | number | ✅ | 経度 |
| `google_maps_place_id` | string | ✅ | Google Places の place_id |
| `google_maps_url` | string | | 検証可能な Maps URL |

### 連絡先

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `phone` | string | | 国際表記。例: `"+84 943 143 686"` |
| `phone_zalo_compatible` | boolean | | 携帯番号で Zalo 利用可能か。`+84 (9|3|5|7|8)x` 始まりなら true |
| `website` | string | | 公式サイト URL |

### 評価・価格

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `rating` | number | | Google Maps の評価（1.0〜5.0） |
| `rating_count` | number | | 評価件数 |
| `price_level_google` | number | | Google の price_level（1〜4） |
| `price_range_vnd_per_person` | [number, number] | ✅ | 1人あたり予算範囲（ドン）。例: `[800000, 1500000]` |

### 料理・タグ

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `cuisine` | string[] | ✅ | 料理ジャンル（複数可）。`tagging-guide.md` の語彙を使う |
| `scenes` | string[] | ✅ | 用途タグ。`business`, `casual-colleagues`, `japanese-guests`, `date`, `family`, `large-group` |
| `ambience` | string[] | | 雰囲気タグ。`fine-dining`, `traditional`, `modern`, `rooftop`, `view`, `garden`, `private-room` |
| `has_view` | boolean | | 景観・夜景があるか |
| `japanese_support` | boolean | | 日本語対応の有無 |

### ミシュラン・ホテル

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `michelin_status` | string \| null | ✅ | `"3-star"` / `"2-star"` / `"1-star"` / `"bib-gourmand"` / `"selected"` / `null` |
| `is_hotel_restaurant` | boolean | ✅ | ホテル併設か |
| `hotel_name` | string | | ホテル名（`is_hotel_restaurant: true` の場合） |

### 予約チャネル

```json
"booking_channels": {
  "phone": "+84 943 143 686",
  "official_website_booking": "https://booking.example.com/...",
  "pasgo_url": "https://pasgo.vn/nha-hang/...",
  "opentable_url": null,
  "tablecheck_url": null,
  "capichi_url": null,
  "facebook_url": "https://facebook.com/...",
  "facebook_messenger": "https://m.me/...",
  "instagram_url": "https://instagram.com/...",
  "zalo_url": "https://zalo.me/84xxx"
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `phone` | string \| null | 予約用電話（基本フィールドの `phone` と同じでよい） |
| `official_website_booking` | string \| null | 公式サイトの予約ページ |
| `pasgo_url` | string \| null | PasGo の店舗ページ |
| `opentable_url` | string \| null | OpenTable の店舗ページ |
| `tablecheck_url` | string \| null | TableCheck の予約ページ |
| `capichi_url` | string \| null | Capichi の店舗ページ |
| `facebook_url` | string \| null | Facebook ページ |
| `facebook_messenger` | string \| null | Messenger 短縮 URL（`m.me/...`） |
| `instagram_url` | string \| null | Instagram |
| `zalo_url` | string \| null | Zalo（携帯番号のみ）|

### メタ情報

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `sources` | string[] | ✅ | 元のソース。`["michelin", "veltra", ...]` |
| `notes` | string | | 自由記述（特徴、口コミ要約、注意事項） |
| `last_updated` | string | ✅ | このエントリの最終更新日（YYYY-MM-DD） |

---

## 完全な例（参考）

```json
{
  "id": "tam-vi",
  "name_official": "Tầm Vị",
  "name_normalized": "tam vi",
  "name_alt": ["タムヴィー"],
  
  "address": "4A Yết Kiêu, Hoàn Kiếm, Hà Nội",
  "district": "Hoàn Kiếm",
  "latitude": 21.0234,
  "longitude": 105.8456,
  "google_maps_place_id": "ChIJxxxx",
  "google_maps_url": "https://maps.google.com/?cid=xxxx",
  
  "phone": "+84 943 143 686",
  "phone_zalo_compatible": true,
  "website": "https://tamvi.vn",
  
  "rating": 4.6,
  "rating_count": 423,
  "price_level_google": 3,
  "price_range_vnd_per_person": [800000, 1500000],
  
  "cuisine": ["Vietnamese", "Modern Vietnamese"],
  "scenes": ["business", "japanese-guests", "date"],
  "ambience": ["traditional", "private-room"],
  "has_view": false,
  "japanese_support": false,
  
  "michelin_status": "1-star",
  "is_hotel_restaurant": false,
  "hotel_name": null,
  
  "booking_channels": {
    "phone": "+84 943 143 686",
    "official_website_booking": null,
    "pasgo_url": null,
    "opentable_url": null,
    "tablecheck_url": null,
    "capichi_url": null,
    "facebook_url": "https://facebook.com/tamvirestaurant",
    "facebook_messenger": "https://m.me/tamvirestaurant",
    "instagram_url": "https://instagram.com/tamvi_restaurant",
    "zalo_url": "https://zalo.me/84943143686"
  },
  
  "sources": ["michelin", "veltra-jp-media"],
  "notes": "ミシュラン1つ星、伝統的なベトナム家庭料理を上質な空間で楽しめる。在住日本人ブログでも接待向けとして頻出。",
  "last_updated": "2026-05-15"
}
```

---

## バリデーションルール

リスト生成時に以下をチェック：

- [ ] `id` がユニーク（重複なし）
- [ ] `name_official` と `address` が空でない
- [ ] `latitude`, `longitude` が妥当な範囲（ハノイ：lat 20.9〜21.1、lng 105.7〜105.9）
- [ ] `cuisine` が `tagging-guide.md` の語彙に従っている
- [ ] `scenes` が `tagging-guide.md` の語彙に従っている
- [ ] `price_range_vnd_per_person[0] >= 500000`（最低でも50万ドン以上）
- [ ] `booking_channels` のURLが妥当な形式
- [ ] `michelin_status` が定義された値のいずれかか `null`
- [ ] `is_hotel_restaurant: true` なら `hotel_name` が必須

不正なエントリは `data/invalid-entries.json` に分けて保存し、人間が確認できるようにする。
