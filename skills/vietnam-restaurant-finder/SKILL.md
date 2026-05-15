---
name: vietnam-restaurant-finder
description: ハノイの中〜高級レストランを `hanoi-restaurants.json`（vietnam-restaurant-list-builder が作ったマスター）から検索・推薦するスキル。ユーザーが「ハノイでレストラン提案して」「今夜どこ行く？」「接待で使える店」「日本人連れて行く店」「フレンチでデート」「ルーフトップで景色いいところ」「ミシュランの店行きたい」「Tay Ho でディナー」などと言ったときに発動する。曖昧な要望（用途・エリア・予算・人数・料理ジャンル）を解釈してランキング上位 3〜5 件を返す。**個別の店の予約手段を取りたい場合は `vietnam-restaurant-booking` を使う**こと。**マスターリスト自体を作る・更新する場合は `vietnam-restaurant-list-builder` を使う**こと。
---

# Vietnam Restaurant Finder（ハノイ中〜高級店レコメンド）

`vietnam-restaurant-list-builder` が作った `hanoi-restaurants.json` を読んで、ユーザーの会食シチュエーションに合う店をランク付けして提示する**日常使い**のスキル。

## このスキルの目的

`vietnam-restaurant-list-builder` が「重い・低頻度・データを作る」のに対し、こちらは「軽い・高頻度・データを使う」側。

毎回 web で検索し直さない。**マスター JSON を信頼してそこから検索する。**

## 発動条件

以下のような会話で必ず使う：

- 「ハノイでレストラン提案して」「どこがいい？」「店を推薦して」
- 「今夜どこで食べる？」「明日の接待どうする？」「奥さんとの記念日に」
- 用途を口頭で指定したとき（接待・デート・日本人ゲスト・大人数）
- 料理ジャンル + ハノイの組み合わせ（「ハノイのフレンチ」「ハノイのミシュラン」など）
- エリア + 用途の組み合わせ（「Tây Hồ で景色いい店」「Hoàn Kiếm で和食」）

「**スキル**」という単語が無くても、ハノイで食事の場面を相談されたら自動的に発動する。

## マスターファイルの場所

```
../vietnam-restaurant-list-builder/data/hanoi-restaurants.json
```

このスキルの SKILL.md から見た相対パス（同じ `skills/` 配下の builder スキルが出力するファイル）。

ファイルが存在しない / 更新が古い場合は `vietnam-restaurant-list-builder` の実行をユーザーに勧める。

## ワークフロー

### Step 1: クエリ解釈

ユーザー発話から以下の軸を抽出する。曖昧で重要な要素が欠けていれば**最大 2 つだけ**確認質問する（それ以上は鬱陶しい）。

| 軸 | 値の例 | 抽出のヒント |
|---|---|---|
| **scenes** | business / date / japanese-guests / family / casual-colleagues / large-group | 「接待」「デート」「記念日」「日本から来客」「同僚と」「家族で」「大人数」 |
| **district** | Hoàn Kiếm / Ba Đình / Tây Hồ / Cầu Giấy / Hai Bà Trưng / Đống Đa / Nam Từ Liêm | エリア名直指定 or 「ホテル近く」「Old Quarter で」 |
| **cuisine** | Vietnamese / French / Japanese / Italian / Chinese / Steakhouse / Fusion 等 | 料理ジャンルそのまま、または「和食」「中華」「フレンチ」「ステーキ」 |
| **予算上限** | `price_range_vnd_per_person[0]` がこれ以下 | 「1 人 100 万ドンまで」「予算 5 千円くらい」（5千円 ≒ 80 万ドン） |
| **ambience** | rooftop / view / private-room / garden / fine-dining / traditional / modern | 「夜景」「景色」「個室」「ガーデン」「カチッと」「ラフな雰囲気」 |
| **人数** | 1-2 / 3-6 / 7+ | 「2 人で」「3 人」「10 名」 — 10+ で `large-group` |
| **michelin 要望** | 1-star / 2-star / 3-star / bib-gourmand / selected / any | 「ミシュランの店」「星付き」「Bib Gourmand で」 |
| **japanese_support** | true / 不問 | 「日本語通じる店」「日本語対応」 |

スキーマ語彙は `references/schema.md`、シーン・雰囲気判定は `references/tagging-guide.md` を参照。

#### 確認質問が必要な典型ケース

ユーザーが「ハノイでレストラン提案して」とだけ言ったら、**用途とエリアの 2 つだけ**確認する：

> どんなシーンですか？（接待 / デート / 日本からの来客 / 同僚と / 家族 / 大人数 など）
> エリアの希望はありますか？（Hoàn Kiếm / Ba Đình / Tây Hồ / Nam Từ Liêm など）

予算や雰囲気を最初から聞くと多くなりすぎる。最初の 5 件を出してから「もっと安く / もっと景色重視」など軌道修正する方が早い。

### Step 2: マスター JSON 読み込みとハードフィルタ

`../vietnam-restaurant-list-builder/data/hanoi-restaurants.json` を読み込み、以下の**ハード条件**で除外する：

1. **district 指定あり** → その区だけ残す（指定なしなら全部）
2. **予算上限指定あり** → `price_range_vnd_per_person[0]` が上限を超える店を除外
3. **cuisine 指定あり** → `cuisine` 配列に少なくとも 1 つ一致する店だけ残す
4. **michelin 要望あり** → `michelin_status` が指定タイプのみ
5. **japanese_support 必須** → `japanese_support: true` のみ

ハード条件で 3 件未満になったら、緩和ステップへ（後述）。

### Step 3: スコアリング

残った各店に以下の合計スコアを付ける：

| 寄与 | 配点 | メモ |
|---|---|---|
| 指定 `scenes` が `r.scenes` に含まれる（1 つあたり） | +5 | 主目的軸なので重い |
| 指定 `ambience` が `r.ambience` に含まれる（1 つあたり） | +3 | |
| `has_view` 一致 | +2 | |
| `japanese_support` true 要望と一致 | +2 | |
| `michelin_status` が星付き (1〜3-star) | +5 | |
| `michelin_status` が bib-gourmand / selected | +3 | |
| `len(sources) >= 3` | +2 | 複数ソースで言及 = 信頼度高 |
| `sources` に "veltra-jp-media" 含む + `japanese-guests` 要望 | +2 | 日本人来客には超強い |
| `is_hotel_restaurant` true + `business` 要望 | +2 | 接待向け |
| `price_range_vnd_per_person[1]` がユーザー予算上限の 70% 以下（コスパ） | +1 | 予算めいっぱい使わなくて済む |

スコアを降順でソート。

### Step 4: 多様性フィルタ

上位 5 件を取る前に、**同じホテル内の店が 2 つを超えないように**調整する。これをやらないと「Capella Hanoi の 4 店」や「Sofitel Metropole の 3 店」が上位を独占して退屈なリストになる。

- ホテル内: `is_hotel_restaurant: true` かつ `hotel_name` 一致を 1 グループとして、最大 2 店だけ残す
- それ以外の店は通常通り

3 番目以降の同ホテル店は「次点候補」として最後に 1 行サマリで紹介してもよい。

### Step 5: 出力

上位 3〜5 件を**マークダウンテーブル + 各店 1〜2 行のコメント**で返す。

#### 通貨表示ルール（重要）

**出力の価格は必ず日本円（JPY）で表示する。** マスター JSON の `price_range_vnd_per_person` は VND だが、ユーザー向け出力時に**JPY に換算してから出す**。

換算レート（このスキル内の規定値）：

```
1 JPY ≒ 160 VND
JPY = VND ÷ 160
```

換算後の丸めルール：

| JPY 範囲 | 表示形式 | 例 |
|---|---|---|
| < 10,000 円 | 100円単位で丸めて `5,000円` | 800,000 VND → `5,000円` |
| ≥ 10,000 円 | 千円単位で丸めて `1.2万円`（1桁小数） | 2,000,000 VND → `1.3万円` |

レート変動を考慮し、**「約」プレフィックスを必ず付ける**（例: `約5,000–9,400円`）。

VND の生値が知りたいユーザーは少数派なので、出力テーブルの主軸列は JPY のみ。VND は最後の「価格レンジの注記」セクションで参考表示するか、ユーザーが明示的に求めたときだけ併記する。

#### 出力フォーマット例

> **今夜の接待 (Hoàn Kiếm + 日本人ゲスト + 予算 1.3万円まで)** — 候補 5 件：
>
> | # | 店名 | 区 | 価格/人 | ミシュラン | 推し理由 |
> |---|---|---|---|---|---|
> | 1 | **Tầm Vị** | Ba Đình | 約5,000–9,400円 | 1★ | VELTRA・在住日本人媒体ともに日本人接待の鉄板、伝統北部料理、個室 |
> | 2 | **Luk Lak** | Hoàn Kiếm | 約4,400–8,100円 | Bib | コロニアル空間で上質なベトナム家庭料理、VELTRA 代行予約対象、Chef Mai |
> | 3 | **Spices Garden** | Hoàn Kiếm | 約7,500–1.6万円 | – | Sofitel Metropole 内、refined Vietnamese、ホテル併設の安心感 |
> | 4 | **Backstage** | Hoàn Kiếm | 約1.3–2.2万円 | Selected | Capella Hanoi、北部 7-course tasting menu、記念会食 |
> | 5 | **Tim Ho Wan (L7)** | Tây Hồ | 約3,100–5,000円 | – | *Hoàn Kiếm 外* — Tây Hồ で軽く済ませる代替案 |
>
> 価格は **1 JPY ≒ 160 VND** で換算した目安。最新レートで±10% 程度ぶれる前提でご覧ください。
>
> **次点**: Hibana by Koki / Izakaya by Koki（同じ Capella Hanoi なので除外）— ハイエンド日本食が刺さるなら直接ご指定ください。
>
> 個別の予約手段（電話・Zalo・公式予約 URL・PasGo URL 等）が必要なら **`vietnam-restaurant-booking` スキルで店名を指定して呼び出してください。**

**重要：嘘をつかない**
- 予算上限を超えている店は「予算上限を一部超過」と明示
- district が外れている店は「Hoàn Kiếm 外」と明示
- `NEEDS_PLACES_API_LOOKUP` フラグ付きの店で距離・最寄駅を聞かれたら「Places API 未連携のため正確な距離は不明」と先に断る

### Step 6: 候補が薄い / 0 件のとき

緩和ステップを順番に適用：

1. **district の隣接区を許容** — 例: Hoàn Kiếm 指定 → Ba Đình / Hai Bà Trưng も含める
2. **予算上限を 20% 緩和** — 「やや高めですが」と注記して候補に含める
3. **cuisine を上位カテゴリに緩和** — 例: "Modern Vietnamese" のみ → "Vietnamese" 全般
4. **それでも 3 件未満** → ユーザーに条件緩和または `vietnam-restaurant-list-builder` でリスト拡張を勧める

### Step 7: フォローアップ

- 「ここの予約方法は？」「電話番号を教えて」→ `vietnam-restaurant-booking` に委譲（店名 / id を渡す）
- 「もっと候補ない？」→ 上位 5 件を出した後の 6〜10 番目を提示、または絞り条件の緩和を提案
- 「Tầm Vị について詳しく」→ マスター JSON の該当 entry を全フィールド整形して返す

## 予算換算メモ（入力・出力で共通）

**換算レート: 1 JPY ≒ 160 VND**（Step 5 の通貨表示ルールと同一）

入出力どちら向きでも同じレートを使う。レート変動で店ごとの所感が変わるので、ユーザーが「今のレート違うんだけど」と言ったら、SKILL 内のレートを上書きして再計算する。

### JPY → VND（ユーザー入力をフィルタ条件に変換）

| ユーザー言い回し | VND 換算（フィルタ閾値） |
|---|---|
| 1 人 3,000 円 | 約 500,000 VND |
| 1 人 5,000 円 | 約 800,000 VND |
| 1 人 10,000 円 | 約 1,600,000 VND |
| 1 人 20,000 円 | 約 3,200,000 VND |

### VND → JPY（マスター JSON の値を出力に変換）

| マスター JSON の値 | 表示する JPY |
|---|---|
| `[500000, 1000000]` | 約 3,100–6,300円 |
| `[800000, 1500000]` | 約 5,000–9,400円 |
| `[1000000, 2500000]` | 約 6,300–1.6万円 |
| `[2000000, 3500000]` | 約 1.3–2.2万円 |
| `[3500000, 6000000]` | 約 2.2–3.8万円 |

レートは大きく変動するので「だいたい」「約」を必ず添える。

## NEEDS_PLACES_API_LOOKUP フラグの扱い

マスター JSON で `notes` フィールドに `NEEDS_PLACES_API_LOOKUP` が含まれる店は、緯度経度が街区レベル近似（Google Places API 未連携）。以下の質問には正確に答えられない：

- 「徒歩 5 分以内のミシュラン店」
- 「2 店間の距離」
- 「最寄り地下鉄駅」

この種のクエリには「**Places API 未連携のため正確な距離・最寄駅は不明。区レベルで提示**」と先に断り、`district` ベースで近接性を提示する。

## 既知の挙動・落とし穴

1. **同一ホテル内の店の独占** — 多様性フィルタで防ぐ（Step 4）。手動で確認すること
2. **`japanese-guests` シーンの強さ** — VELTRA 代行予約対象店は実質的に「日本人来客向け」の最強候補なので、`sources` チェックを必ずする
3. **Bib Gourmand 系の二重キャラ** — `chao-ban` / `the-east-indochine` / `luk-lak` は Michelin 認定だが価格は控えめ。「接待でガッツリ」用途には Le Beaulieu / Backstage 等を、「日本人来客に文化体験」用途には Tầm Vị / Luk Lak / Chao Bản を使い分ける
4. **`michelin_status: null` だがホテル併設の高級店**（Le Beaulieu, Top of Hanoi, French Grill 等）は Michelin 圏外でも一軍評価。「ミシュラン縛り」をユーザーが明示しない限り、これらを上位に入れて OK
5. **`Lamai Garden`** はマスター JSON 上 `michelin_status: null` だが notes に「Michelin Green Star 獲得」と書いてある。Green Star は星付きではないので status には入れない設計。「サステナブル / ヴィーガン / Tây Hồ で良い店」の文脈なら確実に拾う

## このスキルの境界（重要）

- **個別店の予約手段を調べる** → `vietnam-restaurant-booking`
- **マスター JSON を作る・更新する** → `vietnam-restaurant-list-builder`
- **このスキルはマスター JSON からの検索だけ**。web fetch しない、リスト更新しない、店舗情報を補完しない

ユーザーが「この店の電話番号は？」と聞いたら、JSON にあれば返す。無ければ `vietnam-restaurant-booking` を案内する。

## 詳細ドキュメント

- `references/schema.md` — マスター JSON の各フィールドの意味（builder と語彙共有）
- `references/tagging-guide.md` — scenes / ambience の判定基準（builder と語彙共有）
- `../vietnam-restaurant-list-builder/data/hanoi-restaurants.json` — 検索対象本体
- `../vietnam-restaurant-list-builder/data/invalid-entries.json` — 「価格閾値以下で本リストから外したが文化的価値はある」店（Pizza 4P's, Quán Ăn Ngon 等）。**ユーザーが「気軽な店」「観光客向け」を求めたら、ここを補助参照する**
