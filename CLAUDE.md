# ドリンク売上トラッカー — Claude Code 引き継ぎ指示書

## プロジェクト概要

飲食店スタッフが営業中・営業後にレシートを撮影するだけで、
ドリンク売上の分析・蓄積・可視化ができるシングルファイルWebアプリ。

**想定ユーザー**
- 店舗スタッフ（スマホで毎テーブル会計後に使用）
- コンサルタント（PCでデータ確認・分析）

**利用場面**
1. テーブル会計のたびにレシートを撮影 → AIがドリンク内訳を自動抽出
2. 営業終了後に日次締めレシートを撮影 → 日次データ確定
3. レポート画面で週次・月次・曜日別のトレンドを確認

---

## 技術スタック

- **フロントエンド**: React 18（UMD版、CDNから読込）
- **グラフ**: 自前SVGバーチャート（外部依存なし）
- **アイコン**: Tabler Icons（CDN）
- **AI**: Anthropic Claude API（claude-sonnet-4-20250514）/ Vision機能でレシート解析
- **データ保存**: localStorage（キー: `drink_tracker_v2`）
- **APIキー保存**: localStorage（キー: `drink_tracker_key`）
- **ファイル構成**: `index.html` 1ファイルに全コードを収録

### API呼び出しの注意点
ブラウザから直接Anthropic APIを叩くため、以下のヘッダーが必須：
```
'anthropic-dangerous-direct-browser-access': 'true'
```

### GAS Web App プロキシ（クロスデバイス同期）
- `GAS_URL` 定数にデプロイ済みURLを設定
- `postTableToGas(table)` : テーブル保存時にno-corsでGASへPOST（fire-and-forget）
- `GasBulkSync` コンポーネント: 設定タブから既存データを一括同期
- スマホ側は「設定→Googleシートからインポート→取り込む」で同期

---

## データ構造

### localStorage: `drink_tracker_v2`
```json
{
  "tables": [
    {
      "id": "一意ID",
      "date": "2026-05-20",
      "time": "19:45",
      "covers": 4,
      "course": "ディナーコース",
      "drinks": [
        {"name": "生ビール", "qty": 3, "unitPrice": 800, "subtotal": 2400}
      ],
      "drinkTotal": 4800,
      "foodTotal": 12000,
      "total": 16800,
      "drinkPerCover": 1200,
      "drinksPerCover": 1.25,
      "savedAt": "ISO8601"
    }
  ],
  "days": [
    {
      "date": "2026-05-20",
      "totalSales": 450000,
      "totalCovers": 85,
      "drinkSales": 120000,
      "foodSales": 330000,
      "tableCount": 24,
      "drinkPerCover": 1412,
      "drinkRatio": 0.267,
      "closedAt": "ISO8601",
      "notes": ""
    }
  ],
  "measures": {
    "2026-05-20": [
      {
        "id": "一意ID",
        "category": "おすすめドリンク出数",
        "unit": "オーダー",
        "target": 10,
        "actual": 8,
        "productName": "本日のおすすめワイン"
      }
    ]
  }
}
```

---

## 現在の機能

### 「今日」タブ
- [x] テーブルレシート撮影 → Claude Vision でドリンク内訳・人数・コース抽出
- [x] 抽出結果の確認・保存
- [x] 当日のテーブル一覧表示・削除
- [x] 日次締めレシート撮影 → 日次データ確定
- [x] 手入力での日次締め（写真がない場合）
- [x] **具体施策カード** (`MeasuresCard`) — GoalBar直下に表示
  - プリセット: ランチドリンク杯数 / ディナードリンク杯数 / おすすめドリンク出数 / 1人D売上 / テーブル杯数 / その他
  - 「おすすめドリンク出数」選択時は商品名入力欄が追加表示（input + datalist、過去ドリンク名を候補表示）
  - 目標・実績を入力 → 達成率(%) を自動計算・色分け表示（緑≥100% / 橙≥80% / 赤<80%）
  - 複数施策を同日に登録可能
  - データは `data.measures[date]` に保存

### 「レポート」タブ
- [x] 週次・月次・曜日別・期間指定の切り替え
- [x] サマリーKPI（ドリンク売上計・1人あたり・DR比率・日平均）
- [x] SVGバーチャート（1人あたりドリンク売上）
- [x] 日別一覧テーブル
- [x] 期間指定は「この期間で集計」ボタンで確定（pendingRs/pendingRe → rs/re）

### 「設定」タブ
- [x] PIN「9999」入力でアクセス（`adminOpen` state + モーダル）
- [x] `GasBulkSync` コンポーネント（設定タブ最上部）
  - 「一括同期する」: 全テーブルデータをGASへ送信
  - 「重複データを削除」: `date|seat|time|drinkTotal` キーで重複除去

---

## 重要コンポーネント・関数メモ

### MeasuresCard（具体施策）
```
場所: TodayScreen 内、GoalBar直下（dayData/非dayData 両ブランチ）
props: {date:string, data, setData}
state: adding, newCat, newUnit, newTarget, newProductName, customCat, customUnit
データ: data.measures[date] = [{id, category, unit, target, actual, productName?}]
```
**施策プリセットの変更**: `MEASURE_PRESETS` 配列（MeasuresCard関数の直前）を編集。
「おすすめドリンク出数」のみ特別処理あり（商品名フィールド表示 + datalist候補）。

### uniqueTables（重複除去 useMemo）
Reportsコンポーネント内。`date|seat|time` キーで重複を除去。
**`tableDays` / `ldFiltered` / `tablesByDate` はすべて `uniqueTables` を参照すること（`data.tables` 直接参照禁止）。**

```javascript
const uniqueTables=useMemo(()=>{
  const seen=new Set();
  return (data.tables||[]).filter(t=>{
    const key=`${t.date}|${t.seat||''}|${(t.time||'').slice(0,5)}`;
    if(seen.has(key))return false;
    seen.add(key);return true;
  });
},[data.tables]);
```

### GasBulkSync
```
場所: 設定タブ最上部（TableManagerの前）
props: {data, setData}
```
設定タブはPINロック（9999）のため `adminOpen===true` の場合のみ表示。

### PIN Lock（設定タブ）
```javascript
// App() 内
const [adminOpen,setAdminOpen]=useState(false);
const [pinModal,setPinModal]=useState(false);
// タブクリック時: settings かつ !adminOpen → setPinModal(true)
// PIN「9999」一致 → setAdminOpen(true), setTab('settings')
```

### Period確定ボタン（レポートタブ）
```javascript
const [pendingRs,setPendingRs]=useState('');
const [pendingRe,setPendingRe]=useState('');
// 「この期間で集計」ボタン押下時に rs=pendingRs, re=pendingRe をセット
```

---

## ブラッシュアップ候補（優先度順）

### 高優先度
- [ ] **具体施策のレポート連携**: 週次・月次で施策達成率トレンドを可視化
- [ ] **テーブル別コース分析**: コース種別ごとのドリンク消費差を集計・可視化
- [ ] **ドリンクランキング**: よく注文されるドリンク品目トップ5（テーブルデータから集計）
- [ ] **AI改善提案**: 蓄積データをもとに「○曜日はドリンク比率が低い」等のコメント自動生成

### 中優先度
- [ ] **ランチ/ディナー別集計**: 時間帯でセグメント分けした分析
- [ ] **折れ線グラフ追加**: ドリンク比率のトレンド推移
- [ ] **CSVエクスポート**: 日次データをCSVでダウンロード
- [ ] **過去日のデータ入力**: 日付指定で遡って入力できる機能

### 低優先度
- [ ] **複数店舗対応**: 店舗を切り替えてデータを分けて管理
- [ ] **Google Sheets連携**: データをスプレッドシートに自動書き込み
- [ ] **PWA化**: スマホのホーム画面に追加できるようにする

---

## コーディングルール

1. **シングルファイル維持**: `index.html` 1ファイルに全コードを収録すること（ビルド不要）
2. **React UMD**: `import/export` は使わない。`const {useState, useMemo, ...} = React;` 形式
3. **外部ライブラリ最小化**: CDN追加は慎重に。Rechartsは読込失敗の実績があるため使用禁止
4. **日本語UI**: すべてのラベル・メッセージは日本語
5. **モバイル対応**: スマホでも使えるレイアウト（max-width: 760px、タッチ操作考慮）
6. **エラーハンドリング**: API失敗時は赤いエラーメッセージを表示。アプリはクラッシュさせない
7. **localStorage**: データ保存キーは `drink_tracker_v2`、APIキーは `drink_tracker_key`
8. **重複除去**: テーブルデータ集計はすべて `uniqueTables`（useMemo）経由。`data.tables` 直接参照禁止

---

## よく使う修正パターン

### 新しい指標をKPIカードに追加する
`metric-card` クラスのdivを使い、`metric-val` と `metric-lbl` の2要素で構成。

### AI抽出プロンプトを変更する
`callVision()` の第4引数（prompt文字列）を編集。
JSONスキーマを変えた場合は、呼び出し元での `parsed.XXX` 参照も合わせて修正。

### チャートを追加・変更する
`BarChart` コンポーネント（SVGベース）を参照。
`data`（配列）、`valueKey`（値のキー名）、`labelKey`（ラベルのキー名）を渡すだけ。

### 施策プリセットを追加・変更する
`MEASURE_PRESETS` 配列（MeasuresCard関数の直前）を編集。
`{label:'項目名', unit:'単位'}` 形式。
「おすすめドリンク出数」は商品名フィールド・datalist候補の特別処理があるため、
同様の挙動を別項目に追加する場合はフォーム内の条件分岐も修正すること。

---

## 開発・確認方法

```bash
# ローカルで開く（file://でも動作する）
open index.html

# または簡易サーバーで開く
python3 -m http.server 8080
# → http://localhost:8080 にアクセス
```

**GitHub Pages**: https://t-fujiya-sd.github.io/drink-tracker/

**テスト時の注意**: Anthropic APIキーが必要。画面上部の黄色バナーに入力。
