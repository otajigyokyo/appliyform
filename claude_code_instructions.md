# 買出人章管理システム — 再提出通知機能の実装指示

## 概要
会員が書類再提出フォームから書類をアップロードした時に：
1. 事務局（info@jigyokyo.com）に通知メールを自動送信する
2. 管理画面（CardGeneratorUI.html）にアラートバナーを表示する

---

## 【タスク1】GAS側: submitResubmit 関数に通知メール送信を追加

### 対象ファイル
Code.gs（clasp管理のGASプロジェクト内）

### 修正内容
`submitResubmit` 関数内で、不備ステータスを「再提出済み」に更新する行：

```javascript
sheet.getRange(targetRow, colIndex['不備ステータス'] + 1).setValue('再提出済み');
```

この行の**直後**、`return { success: true };` の**前**に、以下のコードを追加してください：

```javascript
    // ========== 事務局へ再提出通知メール送信 ========== //
    try {
      const storeName = String(rowData[colIndex['店名']] || '');
      const wearerName = String(rowData[colIndex['買出人章着用者氏名']] || rowData[colIndex['着用者名']] || '');
      const typeLabel = type === 'photo' ? '顔写真' :
                        type === 'doc' ? '公的書類' : '顔写真および公的書類';

      const subject = '【再提出】' + memberId + ' ' + storeName + ' ' + wearerName + ' 書類再提出がありました';

      const body = '書類の再提出がありました。\n\n'
        + '会員番号: ' + memberId + '\n'
        + '店名: ' + storeName + '\n'
        + '着用者名: ' + wearerName + '\n'
        + '再提出内容: ' + typeLabel + '\n'
        + '再提出日時: ' + Utilities.formatDate(new Date(), 'Asia/Tokyo', 'yyyy/MM/dd HH:mm:ss') + '\n\n'
        + '管理画面で内容をご確認ください。\n'
        + 'https://kaidashi.jigyokyo.com/CardGeneratorUI.html\n\n'
        + '※ このメールはシステムから自動送信されています。';

      GmailApp.sendEmail('info@jigyokyo.com', subject, body, {
        name: '買出人章管理システム'
      });

      Logger.log('再提出通知メール送信完了: ' + memberId);
    } catch (mailError) {
      // メール送信失敗は再提出処理自体のエラーにはしない
      Logger.log('再提出通知メール送信エラー: ' + mailError.toString());
    }
```

---

## 【タスク2】フロント側: CardGeneratorUI.html に再提出アラートバナーを追加

### 対象ファイル
CardGeneratorUI.html（GitHub Pagesリポジトリ内）

### 修正A: HTML追加
`<div class="search-section">` の**直前**に以下を追加：

```html
    <!-- 再提出アラートバナー -->
    <div class="resubmit-alert" id="resubmitAlert" style="display: none;">
      <div class="resubmit-alert-inner">
        <span class="resubmit-alert-icon">🔔</span>
        <span class="resubmit-alert-text" id="resubmitAlertText"></span>
        <button class="resubmit-alert-btn" onclick="loadDeficiencyMembers(1)">確認する</button>
        <button class="resubmit-alert-close" onclick="dismissResubmitAlert()" title="閉じる">✕</button>
      </div>
    </div>
```

### 修正B: CSS追加
`<style>` タグ内の既存スタイルの末尾付近（不備バッジ系CSSの後あたり）に以下を追加：

```css
    .resubmit-alert {
      margin: 0 auto 16px;
      max-width: 1050px;
      animation: slideDown 0.4s ease-out;
    }
    @keyframes slideDown {
      from { opacity: 0; transform: translateY(-10px); }
      to { opacity: 1; transform: translateY(0); }
    }
    .resubmit-alert-inner {
      background: linear-gradient(135deg, #fff3e0, #ffe0b2);
      border: 2px solid #ff9800;
      border-radius: 10px;
      padding: 14px 20px;
      display: flex;
      align-items: center;
      gap: 12px;
      box-shadow: 0 2px 8px rgba(255, 152, 0, 0.2);
    }
    .resubmit-alert-icon {
      font-size: 24px;
      flex-shrink: 0;
    }
    .resubmit-alert-text {
      flex: 1;
      font-size: 14px;
      font-weight: 600;
      color: #e65100;
    }
    .resubmit-alert-btn {
      background: #ff9800;
      color: #fff;
      border: none;
      padding: 8px 18px;
      border-radius: 6px;
      font-size: 13px;
      font-weight: 700;
      cursor: pointer;
      white-space: nowrap;
      transition: background 0.2s;
    }
    .resubmit-alert-btn:hover {
      background: #f57c00;
    }
    .resubmit-alert-close {
      background: none;
      border: none;
      font-size: 18px;
      color: #bf360c;
      cursor: pointer;
      padding: 4px 8px;
      border-radius: 4px;
      flex-shrink: 0;
    }
    .resubmit-alert-close:hover {
      background: rgba(191, 54, 12, 0.1);
    }
```

### 修正C: JavaScript関数追加
`<script>` タグ内の既存関数群の中（例えば `loadDeficiencyMembers` 関数の近く）に以下の2関数を追加：

```javascript
    // ========== 再提出アラートチェック ========== //
    async function checkResubmitAlert() {
      try {
        const result = await callGasApi('getDeficiencyMembers', { page: 1, perPage: 100 });
        if (!result.success) return;

        // 「再提出済み」の会員を数える
        const resubmitted = (result.results || []).filter(
          m => m.deficiencyStatus === '再提出済み'
        );

        const alertEl = document.getElementById('resubmitAlert');
        const textEl = document.getElementById('resubmitAlertText');

        if (resubmitted.length > 0) {
          const names = resubmitted.slice(0, 3).map(
            m => m.memberId + ' ' + m.storeName
          ).join('、');
          const suffix = resubmitted.length > 3
            ? ' 他' + (resubmitted.length - 3) + '件'
            : '';

          textEl.textContent = '📋 書類の再提出があります（'
            + resubmitted.length + '件）：' + names + suffix;
          alertEl.style.display = 'block';
        } else {
          alertEl.style.display = 'none';
        }
      } catch (error) {
        console.log('再提出アラートチェックエラー:', error);
      }
    }

    function dismissResubmitAlert() {
      document.getElementById('resubmitAlert').style.display = 'none';
    }
```

### 修正D: ページ読み込み時にチェックを実行
既存の `DOMContentLoaded` イベントリスナー内、または `</script>` の直前に以下を追加：

```javascript
    // ページ読み込み時に再提出アラートをチェック（1秒後に実行）
    setTimeout(checkResubmitAlert, 1000);
```

※ 1秒遅延させるのは、ページの初期表示を妨げないためです。

---

## 注意事項
- GAS側はコード追加後に clasp push → 新しいバージョンでデプロイが必要
- フロント側はコミット → git push でGitHub Pagesに反映
- 既存の `callGasApi` 関数と `loadDeficiencyMembers` 関数がすでに存在する前提
- メール送信失敗は try-catch で囲んでいるので、再提出処理自体には影響しない
- `@keyframes slideDown` が既存CSSと重複していないか確認すること
