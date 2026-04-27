# テンプレートID採番バグの修正

## バグ概要
メールテンプレートの新規作成・削除時にIDが衝突し、別テンプレートの内容が表示される/別のものが削除される。

## 根本原因
`Code.js` の `saveEmailTemplate` 関数（約2466行目）で、新規テンプレートのIDを `data.length` で生成している：

```javascript
// 🐛 バグ: 削除で行が減るとIDが重複する
const newId = 'TPL' + String(data.length).padStart(3, '0');
```

例: TPL001, TPL002, TPL003 がある状態(data.length=4)でTPL002を削除 → data.length=3 → 次のIDが TPL003 → 既存のTPL003と衝突

## 修正箇所

### Code.js — `saveEmailTemplate` 関数内のID生成部分のみ変更

**変更前（約2466行目）:**
```javascript
const newId = 'TPL' + String(data.length).padStart(3, '0');
```

**変更後:**
```javascript
// 既存IDの最大番号を取得して+1で採番（削除後もIDが重複しない）
let maxNum = 0;
for (let i = 1; i < data.length; i++) {
  const id = String(data[i][0] || '');
  const match = id.match(/^TPL(\d+)$/);
  if (match) {
    const num = parseInt(match[1], 10);
    if (num > maxNum) maxNum = num;
  }
}
const newId = 'TPL' + String(maxNum + 1).padStart(3, '0');
```

## 変更対象ファイル
- `Code.js` のみ（1箇所のみの変更）

## フロントエンド（EmailSender.html）
- 変更不要（フロントエンドはIDをそのまま使っているだけで問題なし）

## デプロイ手順
1. `clasp push`
2. `clasp deploy -i AKfycbwbbyZoRzhWu8Ft5xVvyI1LJc7_RLZaeLwVGapXnlYIIJBUSlgJXQTzofwuLl5nux43kg`
3. ブラウザで `Ctrl+Shift+R` ハードリロード

## テスト手順
1. テンプレートを1つ削除する
2. 新規テンプレートを作成する
3. 作成したテンプレートを開いて、入力した内容が正しく表示されることを確認
4. 別のテンプレートを開いて、内容がズレていないことを確認

## 既存データの注意点
既にIDが重複しているテンプレートがスプレッドシートに存在する可能性がある。
修正デプロイ後、「メールテンプレート」シートを目視確認し、重複IDがあれば手動で片方のIDを変更すること。
