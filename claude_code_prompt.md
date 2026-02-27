# 会費ペイ再登録機能の実装

## 背景
会員登録時に会費ペイAPI連携が失敗し、Sheetには登録されたが会費ペイに未登録の会員がいる。
管理画面から再登録できる機能と、新規登録時のリトライ＆失敗記録機能を追加する。

## 変更対象ファイル
1. KaihipayAPI.gs — 末尾に2関数追加
2. Code.gs — doPostに2 case追加 + saveFormData内の会費ペイ連携部分を修正
3. CardGeneratorUI.html — 会員詳細エリアにボタン2つとJS関数2つ追加

---

## 1. KaihipayAPI.gs — 末尾に以下を追加

```javascript
/**
 * 管理画面から会費ペイ再登録を実行
 * 
 * 処理フロー:
 * 1. スプレッドシートから会員情報を取得
 * 2. 会費ペイに既に登録済みかチェック (GET /customers/{customer_number})
 * 3. 未登録 → 会員登録 → コース追加 → 支払URL生成
 *    登録済み → コース確認 → 支払URL生成
 * 4. スプレッドシートの会費ペイ登録状態を更新
 *
 * @param {Object} params - { memberId: 'K10001' } or { rowNumber: 5 }
 */
function retryKaihipayRegistration(params) {
  Logger.log('=== 会費ペイ再登録開始 ===');
  
  var targetRow = -1;
  var kaihipayStatusIdx;
  
  try {
    var ss = SpreadsheetApp.openById(getSpreadsheetId());
    var sheet = ss.getActiveSheet();
    var data = sheet.getDataRange().getValues();
    var headers = data[0];
    
    var colIndex = {};
    headers.forEach(function(h, i) { colIndex[h] = i; });
    
    // ========== 会員データを取得 ========== //
    var rowData = null;
    
    if (params.memberId) {
      var memberIdIdx = colIndex['member_id'];
      for (var i = 1; i < data.length; i++) {
        if (String(data[i][memberIdIdx] || '').trim() === params.memberId) {
          targetRow = i + 1;
          rowData = data[i];
          break;
        }
      }
    } else if (params.rowNumber) {
      var rowIdx = parseInt(params.rowNumber) - 1;
      if (rowIdx >= 1 && rowIdx < data.length) {
        targetRow = parseInt(params.rowNumber);
        rowData = data[rowIdx];
      }
    }
    
    if (!rowData) {
      return { success: false, error: '会員が見つかりません' };
    }
    
    var memberId = String(rowData[colIndex['member_id']] || '').trim();
    if (!memberId) {
      return { success: false, error: '会員番号がありません' };
    }
    
    Logger.log('対象会員: ' + memberId);
    
    // ========== 会費ペイ登録状態列を確保 ========== //
    kaihipayStatusIdx = colIndex['会費ペイ登録状態'];
    if (kaihipayStatusIdx === undefined) {
      var nextCol = headers.length + 1;
      sheet.getRange(1, nextCol).setValue('会費ペイ登録状態');
      kaihipayStatusIdx = nextCol - 1;
      Logger.log('「会費ペイ登録状態」列を追加しました');
    }
    
    // ========== スプレッドシートから会員情報を構築 ========== //
    var storeName = String(rowData[colIndex['店名']] || '');
    var email = String(rowData[colIndex['メールアドレス']] || '').trim();
    var phone = String(rowData[colIndex['電話']] || '').replace(/^'/, '').replace(/[-\-ー－]/g, '');
    var postalCode = String(rowData[colIndex['郵便番号']] || '').replace(/[-\-ー－]/g, '');
    var address = String(rowData[colIndex['住所']] || '');
    var wearerName = String(rowData[colIndex['買出人章着用者氏名']] || '');
    var wearerNameKana = String(rowData[colIndex['買出人章着用者氏名（ふりがな）']] || '');
    var repName = String(rowData[colIndex['代表者名']] || '');
    var repNameKana = String(rowData[colIndex['代表者名（ふりがな）']] || '');
    
    if (!email) {
      return { success: false, error: 'メールアドレスが登録されていません' };
    }
    
    // ========== 会費ペイに既に登録済みかチェック ========== //
    var alreadyRegistered = false;
    var hasCourse = false;
    
    try {
      var customerInfo = kaihipayRequest('/customers/' + memberId, 'GET', null);
      Logger.log('会費ペイ会員情報取得: ' + JSON.stringify(customerInfo).substring(0, 300));
      
      if (customerInfo.response && customerInfo.response.data) {
        alreadyRegistered = true;
        Logger.log('会費ペイに登録済み: ' + memberId);
        
        var customerData = customerInfo.response.data;
        if (customerData.customer_courses && customerData.customer_courses.length > 0) {
          hasCourse = true;
          Logger.log('コースも追加済み');
        }
      } else if (customerInfo.code === 200 || customerInfo.statusCode === 200) {
        alreadyRegistered = true;
      }
    } catch (checkError) {
      Logger.log('会費ペイ会員確認エラー（未登録と判断）: ' + checkError.toString());
      alreadyRegistered = false;
    }
    
    var config = getKaihipayConfig();
    var stepResults = [];
    
    // ========== ステップ1: 会員登録（未登録の場合のみ） ========== //
    if (!alreadyRegistered) {
      Logger.log('ステップ1: 会員登録を実行');
      
      function splitNameRetry(fullName) {
        if (!fullName) return { last: '', first: '' };
        var name = String(fullName).trim();
        var parts = name.split(/[\s　]+/);
        if (parts.length >= 2) {
          return { last: parts[0], first: parts.slice(1).join('') };
        }
        return { last: name, first: name };
      }
      
      var nameSource = wearerName || repName || '';
      var kanaSource = wearerNameKana || repNameKana || '';
      var kanjiName = splitNameRetry(nameSource);
      var kanaName = splitNameRetry(kanaSource);
      
      var lastNameKana = formatKanaForKaihipay(kanaName.last, 'カイイン');
      var firstNameKana = formatKanaForKaihipay(kanaName.first, 'タロウ');
      
      var memberData = {
        customer_number: memberId,
        last_name: kanjiName.last,
        first_name: kanjiName.first,
        last_name_kana: lastNameKana,
        first_name_kana: firstNameKana,
        mail: email,
        tel: phone,
        zip_code: postalCode,
        address: address
      };
      
      registerKaihipayMember(memberData);
      stepResults.push('会員登録: 成功');
    } else {
      stepResults.push('会員登録: スキップ（登録済み）');
    }
    
    // ========== ステップ2: コース追加（未追加の場合のみ） ========== //
    if (!hasCourse) {
      Logger.log('ステップ2: コース追加を実行');
      addCourseToCustomer(memberId, config.COURSE_ID);
      stepResults.push('コース追加: 成功');
    } else {
      stepResults.push('コース追加: スキップ（追加済み）');
    }
    
    // ========== ステップ3: 認証コード取得 & 支払URL生成 ========== //
    var authCode = getPaymentMethodAuthCode(memberId);
    stepResults.push('認証コード: 取得成功');
    
    var paymentUrl = generatePaymentMethodUrl(authCode);
    stepResults.push('支払URL: 生成成功');
    
    // ========== スプレッドシート更新 ========== //
    var now = Utilities.formatDate(new Date(), 'JST', 'yyyy-MM-dd HH:mm:ss');
    sheet.getRange(targetRow, kaihipayStatusIdx + 1).setValue('登録済み (' + now + ')');
    
    Logger.log('=== 会費ペイ再登録成功 ===');
    
    return {
      success: true,
      memberId: memberId,
      storeName: storeName,
      paymentUrl: paymentUrl,
      steps: stepResults,
      message: memberId + ' (' + storeName + ') の会費ペイ登録が完了しました'
    };
    
  } catch (error) {
    Logger.log('会費ペイ再登録エラー: ' + error.toString());
    
    // エラー時もスプレッドシートに記録
    try {
      if (targetRow > 0 && kaihipayStatusIdx !== undefined) {
        var nowErr = Utilities.formatDate(new Date(), 'JST', 'yyyy-MM-dd HH:mm:ss');
        var ssErr = SpreadsheetApp.openById(getSpreadsheetId());
        var sheetErr = ssErr.getActiveSheet();
        sheetErr.getRange(targetRow, kaihipayStatusIdx + 1).setValue('登録失敗 (' + nowErr + '): ' + error.toString().substring(0, 100));
      }
    } catch (logError) {
      Logger.log('エラー記録失敗: ' + logError.toString());
    }
    
    return {
      success: false,
      error: error.toString(),
      message: '会費ペイ登録に失敗しました: ' + error.toString()
    };
  }
}


/**
 * 会費ペイ登録状態を確認（管理画面用）
 */
function checkKaihipayStatus(params) {
  try {
    var memberId = String(params.memberId || '').trim();
    if (!memberId) {
      return { success: false, error: '会員番号が指定されていません' };
    }
    
    Logger.log('=== 会費ペイ登録状態確認: ' + memberId + ' ===');
    
    var customerInfo = kaihipayRequest('/customers/' + memberId, 'GET', null);
    
    if (customerInfo.response && customerInfo.response.data) {
      var d = customerInfo.response.data;
      var hasCourse = d.customer_courses && d.customer_courses.length > 0;
      var hasPayment = d.payment_method_type && d.payment_method_type !== '';
      
      return {
        success: true,
        registered: true,
        hasCourse: hasCourse,
        hasPaymentMethod: hasPayment,
        customerData: {
          customer_number: d.customer_number,
          name: (d.last_name || '') + ' ' + (d.first_name || ''),
          mail: d.mail || '',
          payment_method_type: d.payment_method_type || '未登録',
          courses: (d.customer_courses || []).map(function(c) {
            return { course_id: c.course_id, course_name: c.course_name || '' };
          })
        }
      };
    }
    
    return {
      success: true,
      registered: false,
      hasCourse: false,
      hasPaymentMethod: false,
      customerData: null
    };
    
  } catch (error) {
    Logger.log('会費ペイ状態確認エラー: ' + error.toString());
    return { success: false, error: error.toString() };
  }
}
```

---

## 2. Code.gs — doPost の switch文に追加

既存の `case 'deleteDuplicate':` の前あたりに以下の2つのcaseを追加:

```javascript
      // ========== 会費ペイ再登録 ========== //
      case 'retryKaihipayRegistration':
        result = executeWithLogging(action, { memberId: requestData.memberId, rowNumber: requestData.rowNumber },
          () => retryKaihipayRegistration(requestData), requestInfo);
        break;

      case 'checkKaihipayStatus':
        result = executeWithLogging(action, { memberId: requestData.memberId },
          () => checkKaihipayStatus(requestData), requestInfo);
        break;
```

---

## 3. Code.gs — saveFormData 内の会費ペイ連携部分を修正

現在のコード（`saveFormData` 関数内）:

```javascript
// ========== 会費ペイ連携 ========== //
    let kaihipayResult = { success: false };
    
    try {
      Logger.log('=== 会費ペイ連携処理開始 ===');
      kaihipayResult = registerMemberAndGetPaymentUrl(formData, memberId);
      Logger.log('会費ペイ連携結果: ' + JSON.stringify(kaihipayResult));
    } catch (kaihipayError) {
      Logger.log('会費ペイ連携エラー: ' + kaihipayError.toString());
      // 会費ペイエラーでもフォーム送信は成功させる
      kaihipayResult = { success: false, error: kaihipayError.toString() };
    }
```

これを以下に置き換え:

```javascript
// ========== 会費ペイ連携（リトライ付き） ========== //
    let kaihipayResult = { success: false };
    
    try {
      Logger.log('=== 会費ペイ連携処理開始 ===');
      kaihipayResult = registerMemberAndGetPaymentUrl(formData, memberId);
      Logger.log('会費ペイ連携結果: ' + JSON.stringify(kaihipayResult));
      
      // 失敗時: 3秒待って1回リトライ
      if (!kaihipayResult.success) {
        Logger.log('会費ペイ登録失敗 → 3秒後にリトライ');
        Utilities.sleep(3000);
        kaihipayResult = registerMemberAndGetPaymentUrl(formData, memberId);
        Logger.log('リトライ結果: ' + JSON.stringify(kaihipayResult));
      }
    } catch (kaihipayError) {
      Logger.log('会費ペイ連携エラー: ' + kaihipayError.toString());
      
      // 1回リトライ
      try {
        Logger.log('例外発生 → 3秒後にリトライ');
        Utilities.sleep(3000);
        kaihipayResult = registerMemberAndGetPaymentUrl(formData, memberId);
        Logger.log('リトライ結果: ' + JSON.stringify(kaihipayResult));
      } catch (retryError) {
        Logger.log('リトライも失敗: ' + retryError.toString());
        kaihipayResult = { success: false, error: retryError.toString() };
      }
    }
    
    // ★ 会費ペイ登録状態をスプレッドシートに記録
    try {
      const currentHeaders = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
      let kaihipayColIdx = currentHeaders.indexOf('会費ペイ登録状態');
      
      if (kaihipayColIdx === -1) {
        const nextCol = currentHeaders.length + 1;
        sheet.getRange(1, nextCol).setValue('会費ペイ登録状態');
        kaihipayColIdx = nextCol - 1;
      }
      
      const lastRow = sheet.getLastRow();
      const nowStr = Utilities.formatDate(new Date(), 'JST', 'yyyy-MM-dd HH:mm:ss');
      
      if (kaihipayResult.success) {
        sheet.getRange(lastRow, kaihipayColIdx + 1).setValue('登録済み (' + nowStr + ')');
      } else {
        const errorMsg = (kaihipayResult.error || '不明なエラー').substring(0, 100);
        sheet.getRange(lastRow, kaihipayColIdx + 1).setValue('未登録 (' + nowStr + '): ' + errorMsg);
      }
    } catch (statusError) {
      Logger.log('会費ペイ登録状態の記録エラー: ' + statusError.toString());
    }
```

---

## 4. CardGeneratorUI.html — ボタンとJS追加

### ボタンHTML
会員詳細表示エリアのボタン群（カード生成ボタン等がある付近）に追加:

```html
<button id="btnRetryKaihipay" onclick="retryKaihipayRegistration()" 
  style="background: linear-gradient(135deg, #ff6b35 0%, #f7931e 100%); 
         color: white; border: none; padding: 10px 20px; border-radius: 6px; 
         cursor: pointer; font-weight: bold; margin: 4px;
         box-shadow: 0 2px 8px rgba(255,107,53,0.3);">
  💳 会費ペイ再登録
</button>

<button id="btnCheckKaihipay" onclick="checkKaihipayStatus()" 
  style="background: linear-gradient(135deg, #17a2b8 0%, #138496 100%); 
         color: white; border: none; padding: 10px 20px; border-radius: 6px; 
         cursor: pointer; font-weight: bold; margin: 4px;">
  🔍 会費ペイ状態確認
</button>
```

### JavaScript関数
`<script>` セクション内に追加:

```javascript
/**
 * 会費ペイ再登録を実行
 */
async function retryKaihipayRegistration() {
  if (!selectedMemberData || !selectedMemberData.memberId) {
    alert('会員が選択されていません');
    return;
  }
  
  var memberId = selectedMemberData.memberId;
  var storeName = selectedMemberData.storeName || '';
  
  if (!confirm(
    memberId + ' ' + storeName + ' の会費ペイ登録を実行しますか？\n\n' +
    '処理内容:\n' +
    '1. 会費ペイに会員登録（未登録の場合）\n' +
    '2. コース追加（未追加の場合）\n' +
    '3. 支払い案内メールが会員に自動送信されます'
  )) {
    return;
  }
  
  var btn = document.getElementById('btnRetryKaihipay');
  var originalText = btn.textContent;
  btn.disabled = true;
  btn.textContent = '⏳ 処理中...';
  btn.style.opacity = '0.6';
  
  try {
    var response = await fetch(GAS_API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        action: 'retryKaihipayRegistration',
        memberId: memberId
      })
    });
    
    var result = await response.json();
    
    if (result.success) {
      var stepsText = (result.steps || []).join('\n');
      alert(
        '✅ 会費ペイ登録成功！\n\n' +
        '会員番号: ' + result.memberId + '\n' +
        '店名: ' + result.storeName + '\n\n' +
        '処理結果:\n' + stepsText + '\n\n' +
        '会員に支払い案内メールが送信されます。'
      );
      
      // 会員詳細を再読み込み
      if (typeof loadMemberDetail === 'function') {
        loadMemberDetail(selectedMemberData.rowNumber);
      }
    } else {
      alert('❌ 会費ペイ登録に失敗しました\n\n' + (result.error || result.message || '不明なエラー'));
    }
    
  } catch (error) {
    alert('❌ 通信エラー: ' + error.message);
  } finally {
    btn.disabled = false;
    btn.textContent = originalText;
    btn.style.opacity = '1';
  }
}


/**
 * 会費ペイ登録状態を確認
 */
async function checkKaihipayStatus() {
  if (!selectedMemberData || !selectedMemberData.memberId) {
    alert('会員が選択されていません');
    return;
  }
  
  var memberId = selectedMemberData.memberId;
  
  var btn = document.getElementById('btnCheckKaihipay');
  var originalText = btn.textContent;
  btn.disabled = true;
  btn.textContent = '🔍 確認中...';
  
  try {
    var response = await fetch(GAS_API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        action: 'checkKaihipayStatus',
        memberId: memberId
      })
    });
    
    var result = await response.json();
    
    if (result.success) {
      if (result.registered) {
        var cd = result.customerData || {};
        var courses = (cd.courses || []).map(function(c) { return c.course_name || c.course_id; }).join(', ') || 'なし';
        
        alert(
          '✅ 会費ペイに登録済み\n\n' +
          '会員番号: ' + cd.customer_number + '\n' +
          '氏名: ' + cd.name + '\n' +
          'メール: ' + cd.mail + '\n' +
          '支払方法: ' + cd.payment_method_type + '\n' +
          'コース: ' + courses + '\n\n' +
          (result.hasPaymentMethod 
            ? '支払方法も登録されています。' 
            : '⚠️ 支払方法が未登録です。会員に案内メールが届いているか確認してください。')
        );
      } else {
        alert(
          '❌ 会費ペイに未登録\n\n' +
          memberId + ' は会費ペイに登録されていません。\n' +
          '「💳 会費ペイ再登録」ボタンで登録を実行してください。'
        );
      }
    } else {
      alert('❌ 確認に失敗しました\n\n' + (result.error || '不明なエラー'));
    }
    
  } catch (error) {
    alert('❌ 通信エラー: ' + error.message);
  } finally {
    btn.disabled = false;
    btn.textContent = originalText;
  }
}
```

---

## 確認事項
- CardGeneratorUI.html 内で `GAS_API_URL` と `selectedMemberData` という変数名が使われているか確認し、実際の変数名に合わせて修正すること
- `loadMemberDetail` 関数が存在するか確認し、なければ会員詳細再読み込みの該当関数名に置き換えること
- clasp push でデプロイ後、新しいデプロイを作成すること（GASのWebアプリURL更新が必要）
