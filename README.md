# My Claude Skills & Plugins

This repository is a Claude Code plugin marketplace that bundles my custom skills as a single plugin.

## Install

Claude Code から marketplace を追加し、`my-claude-skills` プラグインを install する:

```text
/plugin marketplace add mygungnir/my_claude_skills
/plugin install my-claude-skills@my-claude-skills
```

更新する場合:

```text
/plugin marketplace update my-claude-skills
```

## Skills

### Language learning
- `skills/vietnamese-study/` - ベトナム語学習の音声教材作成スキル

### Hanoi restaurants (3-skill set)

ハノイの中〜高級レストランを「集める → 検索する → 予約する」で分業する 3 スキル構成。

- `skills/vietnam-restaurant-list-builder/` - マスター JSON `hanoi-restaurants.json` を作成・更新する低頻度・重いスキル。Michelin Guide / 5 つ星ホテル / PasGo / VELTRA 等の権威ソースから店舗を集約し、Phase 1〜4（集約・エンリッチ・マージ・タグ付け）で 100〜150 店舗を構築する。初期データとして 40 店舗を `data/hanoi-restaurants.json` に同梱。
- `skills/vietnam-restaurant-finder/` - マスター JSON から日常的に店を検索・推薦するスキル。シーン / エリア / 予算 / 料理 / 雰囲気 / 人数 / Michelin 要望 / 日本語対応 を抽出してランキング上位 3〜5 件を返す。価格は **JPY** 表示（1 JPY ≒ 160 VND で換算）。
- `skills/vietnam-restaurant-booking/` - 単一店の予約導線（電話 / Zalo / Facebook Messenger / 公式予約フォーム / PasGo / OpenTable / VELTRA 代行）を返すスキル。web 補完で新発見した `booking_channels` はマスター JSON に **in-place で追記**する（safe-write: `null` のフィールドだけ埋め、non-null は触らない）。

