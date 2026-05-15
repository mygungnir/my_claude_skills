# Sources（ソース別 収集手順）

7つのソースについて、それぞれの収集手順と注意点。

---

## 1. Michelin Guide Hanoi

**URL:** `https://guide.michelin.com/vn/en/ha-noi/ha-noi_2974158/restaurants`

**カテゴリ別ページ：**
- 星付き全体: `.../restaurants/all-starred`
- ビブグルマン: `.../restaurants/bib-gourmand`

**収集手順（公式トライ）：**
1. `web_fetch` で一覧ページを取得
2. 各店舗のリンクは `/vn/en/ha-noi/ha-noi_xxx/restaurant/<slug>` 形式
3. 各店舗ページに住所・電話・料理ジャンル・価格レンジ（₫₫₫ 等）が記載
4. `michelin_status` フィールドに以下のいずれかをセット：
   - `"3-star"` / `"2-star"` / `"1-star"` / `"bib-gourmand"` / `"selected"`

**⚠️ 公式が 403 だったときのフォールバック（実運用ではほぼ毎回こちら）：**

`guide.michelin.com` は WebFetch を bot ブロックして 403 を返すことがほとんど。**1 回試して失敗したら即フォールバック**に切り替える（リトライしない）。

フォールバックフロー：

1. **Wikipedia 横断ソース**：`web_fetch https://en.wikipedia.org/wiki/List_of_Michelin-starred_restaurants_in_Vietnam`
   - 星付き（1-star / 2-star / 3-star）の一次情報源として最も安定
   - 年次（2023 / 2024 / 2025）も書かれているので、直近版だけを採用する

2. **Michelin 自身のニュース記事（リダイレクトが効くことがある）**：
   - `web_fetch https://guide.michelin.com/en/article/michelin-guide-ceremony/full-list-michelin-stars-michelin-guide-vietnam-2025`
   - `…/ae-du/en/article/…` `…/ae-az/en/article/…` のようにロケール違いの記事 URL は別ホスト扱いで通ることがある

3. **旅行メディア / 地元グルメメディア（住所・電話の補完源）**：
   - `bestpricetravel.com` のハノイ Michelin まとめは住所・電話まで載っていて精度が高い
   - `hafoodtours.com` `hanoivoyage.com` などのキュレーション記事
   - これらを 2 〜 3 本クロス参照して矛盾がないか確認する

4. **三角測量の判定ルール**：
   - 「2 ソース以上で一致する店」だけを `raw-michelin.json` に採用
   - 1 ソースしか言及していない店は `raw_notes` に `"single_source: <source_name>"` を残して採用は保留
   - **年次の食い違い**（2024 で Bib Gourmand → 2025 で外れた等）は最新年を採用、過去版の情報は `notes` に履歴として残す

5. **2026 版発表後のリフレッシュ**：
   - Michelin Vietnam 2026 セレモニーは 2026-06-04
   - これ以降に再実行する場合、Wikipedia の反映が遅いので、上記 2 の article URL から 2026 版を取りに行く
   - 2025 版と 2026 版で `michelin_status` が変わった店は `notes` に履歴を残す

**注意：**
- ミシュランは「Selected」（無冠掲載）も価値あるので必ず含める
- ページネーション要確認（30件以上ある場合）
- Wikipedia は **星付きしか網羅していない**（Bib Gourmand / Selected は薄い）。Bib Gourmand / Selected は手順 3 の旅行メディアを主ソースに据える
- フォールバック経由で取った店は `raw-michelin.json` の `sources_chain` に「公式が 403 だったため Wikipedia + bestpricetravel で三角測量」と必ず明記する

**期待件数：** 30〜50店舗（2025年版ベース）

---

## 2. OpenTable Hanoi

**URL:** `https://www.opentable.com/metro/vietnam` から Hanoi で絞り込み

または `https://www.opentable.com/r/hanoi` 形式の都市別ページを探す

**収集手順：**
1. ハノイ全店リストのページを `web_fetch`
2. 各店舗カードから店名・住所・cuisine・price band（$$, $$$, $$$$）を抽出
3. OpenTable に掲載されている時点で「予約可能な中〜高級店」と判定してよい
4. `booking_channels.opentable_url` を必ず保存

**注意：**
- ホテル併設レストランが多く含まれる（後でホテル判定）
- $（一番安い）は除外、$$ 以上のみ採用

**期待件数：** 30〜50店舗

---

## 3. PasGo 高級カテゴリ

**URL:** `https://pasgo.vn/ha-noi`

**フィルタ：**
- カテゴリ「Cao cấp」（高級）、または「Sang trọng」（豪華）
- 価格レンジで絞り込み（500,000 VND/人以上）
- 料理ジャンル別ページ：
  - フレンチ: `pasgo.vn/ha-noi/nha-hang/am-thuc-phap`
  - 日本料理: `pasgo.vn/ha-noi/nha-hang/am-thuc-nhat-ban`
  - イタリアン: `pasgo.vn/ha-noi/nha-hang/am-thuc-y`
  - シーフード: `pasgo.vn/ha-noi/nha-hang/hai-san`

**収集手順：**
1. 各カテゴリページを `web_fetch`
2. 店舗カードから店名・住所・価格レンジ・PasGoのURL を抽出
3. `booking_channels.pasgo_url` を保存

**注意：**
- PasGo は「中級」が多めなので、価格レンジで再フィルタする
- 80万ドン未満の店は除外する判断を入れる

**期待件数：** 50〜100店舗（フィルタ後）

---

## 4. TimeOut Hanoi - Best Restaurants

**URL検索：**
- `web_search` で「TimeOut Hanoi best restaurants 2025」「TimeOut Hanoi fine dining」
- TimeOut Hanoi の公式記事を見つけて `web_fetch`

**よく出てくる記事タイトル：**
- "The best restaurants in Hanoi"
- "Hanoi's best fine dining"
- "Best date night restaurants in Hanoi"

**収集手順：**
1. 編集者選の記事を1〜3本探す
2. 各記事から店名と説明文を抽出
3. 記事の評価コメントを `raw_notes` に保存（タグ付けの材料）

**注意：**
- TimeOut の記事は2〜3年前のものもあるので公開日を確認
- 閉店している店もあるので、エンリッチ時に営業確認

**期待件数：** 20〜40店舗

---

## 5. VELTRA / KKday の代行予約対象店

**URL：**
- VELTRA: `https://www.veltra.com/jp/asia/vietnam/hanoi/a/186726`（ハノイ代行予約サービス）
- KKday: `https://www.kkday.com/ja/category/vn-vietnam/restaurants/list`

**収集手順：**
1. VELTRA の代行予約サービスページを `web_fetch`
2. 「予約可能店舗の一例」セクションから店名・住所・電話を抽出
3. KKday の場合は個別商品ページから対象店を抽出
4. `sources` に "veltra" / "kkday" を追加

**注意：**
- VELTRA は「例」として店名を載せているだけで、実際は「どの店でも代行予約可能」
- ただし「例」として載っている店は、日本人が頻繁に予約する店 = 接待・観光に使える店

**期待件数：** 10〜20店舗

---

## 6. 在住者向け日本語メディア

**主要メディア：**
- **ベトナムリアルガイド** (hataraku-mama.info)：「ハノイ おすすめレストラン」「接待」記事
- **Poste** (poste-vn.com)：在住日本人向け
- **ベトナムニュース・VIETJO**：レストラン特集
- **ベトナムスケッチ**：在住日本人雑誌のオンライン版
- **TNK Travel ブログ**：旅行会社のキュレーション

**検索クエリ例：**
```
ハノイ 接待 レストラン おすすめ
ハノイ 高級 レストラン 日本人
ハノイ デート 記念日 レストラン
```

**収集手順：**
1. `web_search` でこれらのメディアを横断検索
2. キュレーション記事を3〜5本見つけて `web_fetch`
3. 各記事から店名を抽出（記事の評価コメントは `raw_notes` に）

**注意：**
- 「日本人在住者がリピートする店」 = `scenes: ["business", "japanese-guests"]` の判定材料
- 日本食店が含まれる場合、`japanese_support: true` をデフォルトでセット

**期待件数：** 20〜40店舗

---

## 7. 5つ星ホテル併設レストラン

**主要ホテル：**

| ホテル | 主なレストラン |
|---|---|
| Sofitel Legend Metropole | Le Beaulieu / Spices Garden / Angelina / La Terrasse |
| Lotte Hotel Hanoi | Top of Hanoi / Boribana / Quan An Ngon at Lotte |
| Hilton Hanoi Opera | Chez Manon |
| JW Marriott Hanoi | French Grill / JW Cafe / Akira Back |
| InterContinental Hanoi Westlake | Saigon Lounge / Cafe Du Lac / Milan |
| Sheraton Hanoi | Oven D'Or / Hemispheres |
| Pan Pacific Hanoi | Pacifica / Ming Restaurant |
| Hanoi Daewoo | Silk Road / Edo Japanese / La Paix |
| Capella Hanoi | Backstage / Hudson Rooms / Diva's Lounge |

**収集手順：**
1. 各ホテルの公式サイトの「Dining」「Restaurants」ページを `web_fetch`
2. 各レストランの紹介ページから情報抽出
3. `is_hotel_restaurant: true`、`hotel_name` をセット
4. 屋上・最上階の店は `ambience: ["view"]` または `["rooftop"]`

**注意：**
- 「Top of Hanoi」のような夜景重視のレストランは確実に拾う
- 同じホテル内に複数レストランがある場合、それぞれ別エントリに

**期待件数：** 20〜30店舗

---

## 追加検討ソース（必要に応じて）

### Asia's 50 Best Restaurants

ベトナムからは毎年数店舗ランクイン。Hanoiは少ないがホーチミンの店はランクイン例あり。  
**URL:** `https://www.theworlds50best.com/asia/en/the-list.html`

### Vietcetera Hanoi

英語ベトナム情報メディア。Modern Vietnamese や Fine Dining の特集記事多数。  
**URL検索:** `web_search` で「Vietcetera Hanoi fine dining」

### Tripadvisor Hanoi Fine Dining

評価でフィルタ可能だが、観光客バイアスが強いので参考程度。優先度低い。

---

## 全ソース横断時の重複対応

複数ソースに掲載されている店は**ヒット率が高い = 信頼度が高い**ので、`sources` 配列に全ソースを残す：

```json
"sources": ["michelin", "opentable", "veltra-jp-media", "hotels"]
```

`vietnam-restaurant-finder` で「ソースが3つ以上の店 = 一軍」のような重み付けに使える。
