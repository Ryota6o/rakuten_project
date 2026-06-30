# 楽天同一商品比較アプリ — ファイル構成

| 項目 | 内容 |
| --- | --- |
| 対象 | 楽天市場 同一商品 横断比較アプリ（フェーズ1：Webアプリ → フェーズ2：Chrome拡張） |
| 作成日 | 2026-06-27 |
| 関連 | 仕様書 v1.2 |

---

## 1. 設計の原則

1. **中核を独立させる**：`pipeline/`（エンティティ解決）と `eval/`（評価）を独立フォルダにし、面接で問われる中核＝自分が握る層を物理的に分離する。`backend/` はこれを import するだけの薄い層に保つ。
2. **評価を一級市民にする**：`eval/metrics.py` が名寄せのテストハーネスを兼ねる。評価をコードの片隅でなく独立フォルダに置く。
3. **役割で色分け（ディレクトリで分離）**：中核（自分）／配管（AI主導）／フェーズ2／データ・設定 を明確に分ける。
4. **実行時の呼び出し方向を一方向に**：`frontend / extension → backend → pipeline`。楽天APIを叩くのは `pipeline/ingest`（＝`scripts/run_ingest.py` のバッチ）のみ。ユーザー検索時は `backend` が `data/` のDBを読み、楽天を叩かない（1 QPS制約に縛られない設計）。

---

## 2. ディレクトリツリー

```
rakuten-price-compare/
├── README.md                  # 指標・アブレーション・制約・設計判断（面接資料兼用）
├── .gitignore                 # data/ と .env を除外
├── .env.example               # 鍵の雛形（実体 .env は除外）
├── pyproject.toml             # 依存管理（uv / conda）
│
├── pipeline/                  ★中核：自分が握る層
│   ├── __init__.py
│   ├── ingest/                # データ取得
│   │   ├── rakuten_client.py  # API呼び出し＋スロットリング(1.5s)、accessKey/IP対応
│   │   └── fetch_items.py     # カテゴリ単位の取り込み（最大3,000件/クエリ）
│   ├── normalize/             # 正規化・型番/属性抽出
│   │   ├── clean.py           # ノイズ除去（【送料無料】等）
│   │   └── extract.py         # 型番・ブランド・容量の抽出
│   ├── blocking/              # 候補ペア生成
│   │   └── blocker.py         # ブロックキー＋FAISS近傍
│   ├── matching/              # 一致判定
│   │   ├── features.py        # 語彙・意味・型番の特徴量
│   │   └── classifier.py      # LightGBM等で一致確率
│   ├── clustering/            # 同一商品の統合
│   │   └── cluster.py         # 連結成分＋閾値
│   ├── pricing/               # 実質価格の算出
│   │   └── effective_price.py # 価格＋送料フラグ−ポイント簡易
│   ├── config.py              # 閾値・定数を一元管理（説明できるように）
│   └── pipeline.py            # 各段を繋ぐオーケストレーション
│
├── eval/                      ★評価：差別化の柱
│   ├── labels/                # 手動正解ラベル(200–500ペア) + アノテーション定義
│   ├── make_eval_set.py       # ラベル作成・管理
│   └── metrics.py             # precision/recall/F1・アブレーション
│
├── backend/                   # FastAPI（pipelineを呼ぶ薄いAPI層）
│   ├── app/
│   │   ├── main.py            # エントリ
│   │   ├── api/               # ルーター（/search, /clusters …）
│   │   ├── schemas/           # Pydantic（レスポンス契約）
│   │   └── deps.py
│   └── tests/                 # API層のテスト（AIに書かせる）
│
├── frontend/                  # React + Vite（比較テーブルUI）
│   ├── src/
│   │   ├── components/        # 比較テーブル等
│   │   ├── api/               # backend呼び出し
│   │   └── App.tsx
│   ├── index.html
│   └── package.json
│
├── extension/                 # フェーズ2：Chrome拡張（Manifest V3）
│   ├── manifest.json
│   ├── content/               # 閲覧中商品の特定＋オーバーレイ
│   ├── background/
│   └── popup/
│
├── scripts/
│   └── run_ingest.py          # 収集バッチ（cron / 手動実行）
│
├── data/                      # gitignore対象
│   ├── raw/                   # API生レスポンス
│   ├── processed/             # 正規化・中間生成物
│   └── app.db                 # SQLite / DuckDB
│
└── docs/
    └── 仕様書.md
```

---

## 3. ディレクトリ別の責務と所有

| ディレクトリ | 責務 | 主担当 |
| --- | --- | --- |
| `pipeline/` | エンティティ解決の全工程（中核ロジック） | **自分**（設計判断・アルゴリズム） |
| `eval/` | 評価セット・指標・アブレーション | **自分**（評価設計・ラベル付け） |
| `backend/` | pipelineを呼ぶFastAPI（薄いAPI層） | AI主導（自分は応答契約を設計） |
| `frontend/` | React比較テーブルUI | AI主導（自分は見せ方を設計） |
| `extension/` | Chrome拡張（フェーズ2の表示層） | AI主導（自分は商品特定→照合を設計） |
| `scripts/` | 収集バッチの実行口 | AI主導 |
| `data/` | DB・中間生成物（gitignore） | — |
| `docs/` | 仕様書・補助ドキュメント | 自分 |

---

## 4. 実行時の呼び出し方向

```
[利用者]
   │ 検索
   ▼
frontend / extension ──▶ backend ──▶ pipeline ──▶ (data/ のDBを読む)
                                          │
                                          └─ ingest だけが楽天APIを叩く
                                             （scripts/run_ingest.py のバッチ）
```

- ユーザー検索のたびに楽天を叩かない。叩くのは収集バッチのみ → レート制限（1 QPS）に縛られない。
- 価格は変動するため、バッチの更新頻度と「◯時点の価格」表示の方針を別途決める（仕様書 §15.3）。

---

## 5. 鍵・データの管理

- `applicationId` / `accessKey` は `.env` に置き、`.gitignore` でリポジトリから除外する。`.env.example` に雛形のみコミットする。
- `data/`（生レスポンス・DB）も `.gitignore` 対象。リポジトリには再現用のスクリプトと少量サンプルのみ。
- AI生成コードが鍵を直書きしていないか、コミット前に必ず確認する。

---

## 6. 構築の順序（ウォーキングスケルトン）

W1では**このツリー全体を最初に作り、各ファイルはほぼ空のスタブ**にしておく。そのうえで週ごとに `normalize` → `blocking` → `matching` → `clustering` → `pricing` と中身を本実装へ差し替える。骨組みを先に通すことで、統合・デプロイの不確実性を初週に解消する。各週の詳細は「進め方（W1–W10）」を参照。
