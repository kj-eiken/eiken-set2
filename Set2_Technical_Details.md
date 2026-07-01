# 英検準2級演習アプリ Set2 実装詳細分析 ・ 技術ドキュメント

**作成日**: 2026年7月1日  
**対象**: GitHub kj-eiken/eiken-set2  
**バージョン**: 1.0

---

## 🏗️ アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────┐
│                    フロントエンド (HTML/JS)              │
│  - ログイン画面                                          │
│  - ホーム画面（大問カード）                              │
│  - 大問1〜6（筆記問題）                                   │
│  - リスニング（第1〜3部）                                │
│  - 採点結果表示                                          │
└──────────────────┬──────────────────────────────────────┘
                   │ fetch / JSONP
                   ↓
┌─────────────────────────────────────────────────────────┐
│              バックエンド (Google Apps Script)           │
│  - ログイン認証 (handleLogin)                            │
│  - 問題データ取得 (handleGetQuestions)                   │
│  - 採点・スコア計算 (checkAnswers)                       │
│  - メール送信 (handleSend)                              │
│  - スプレッドシート記録 (logResults)                     │
└──────────────────┬──────────────────────────────────────┘
                   │ スプレッドシート API
                   ↓
        ┌──────────────────────────────┐
        │  Google スプレッドシート      │
        │  - Sheet1 (学生リスト)       │
        │  - 結果集計_Set2（結果記録） │
        └──────────────────────────────┘
                   │ Gmail API
                   ↓
        ┌──────────────────────────────┐
        │      Gmail (メール送信)       │
        │  宛先: online-eigojuku...    │
        └──────────────────────────────┘
```

---

## 📦 フロントエンド (HTML/JS) 実装詳細

### 1. ページ構成

#### ログイン画面 (`#login-overlay`)
```html
<div class="login-overlay" id="login-overlay">
  <div class="login-box">
    <input id="login-id" class="login-input" placeholder="例：00000001">
    <input id="login-name" class="login-input" placeholder="氏名を入力">
    <button onclick="doLogin()">ログイン</button>
  </div>
</div>
```

**実装内容**:
- 学生ID (8桁): `login-id`
- 氏名: `login-name`
- ログイン認証: `doLogin()` → GAS `handleLogin()` を呼び出し
- 期限チェック: 2027年1月25日以後はログイン不可

#### ホーム画面 (`#page-home`)
```html
<div class="page active" id="page-home">
  <!-- 進捗バー -->
  <div class="prog-wrap"><div class="prog-fill" id="total-bar"></div></div>
  
  <!-- 大問カード -->
  <div class="sec-cards">
    <div class="sec-card" onclick="goPage('q1')">
      <div class="sc-num">1</div>
      <div class="sc-title">語彙・文法・熟語</div>
      <div class="sc-count">4択 × 15</div>
    </div>
    <!-- ... 以降、大問2〜6 ... -->
  </div>
  
  <!-- リスニングカード -->
  <div class="sec-card listening-card">
    <div class="sc-num listening-num">🎧</div>
    <div class="sc-title">リスニング（第1部〜第3部）</div>
  </div>
</div>
```

**実装内容**:
- 進捗バー: `total-bar` (CSS width アニメーション)
- 大問カード: 各大問の状態を表示
  - スコア表示: `#home-score-q1` など
  - 「送信するとスコアが表示」メッセージ
- リスニングカード: 第1〜3部のスコア表示

#### 大問ページ

**大問1 (`#page-q1`)**
```html
<div class="page" id="page-q1">
  <div class="sec-hdr"><h2>大問1: 語彙・文法・熟語</h2></div>
  <div id="q1-container"></div>  <!-- 動的に問題を挿入 -->
  <button class="submit-btn" id="q1-submit" onclick="checkSection('q1')">解答する</button>
  <div class="result-area" id="q1-result"></div>  <!-- 採点結果表示 -->
</div>
```

**実装フロー**:
1. ページ読み込み → GAS から問題データ取得
2. `#q1-container` に問題HTML を動的生成
3. ユーザーが選択肢を選択 → `answers['q1-1']` など に記録
4. 「解答する」 → `checkSection('q1')` 実行
5. 採点結果を `#q1-result` に表示

**大問2〜4** も同じパターン

**大問5・6** (英作文):
```html
<div class="page" id="page-q5">
  <!-- モデル解答・採点ポイント表示 -->
  <div class="model-answer">...</div>
  <div class="points-list">...</div>
  
  <!-- テキスト入力 -->
  <textarea id="q5-text" placeholder="40〜50語で作成してください"></textarea>
  
  <!-- 送信ボタン -->
  <button onclick="submitAnswers()">送信</button>
</div>
```

#### リスニング (`#page-listening`)
```html
<div class="page" id="page-listening">
  <div id="listening-questions"></div>  <!-- 動的に問題を挿入 -->
  
  <!-- 各問の構成 -->
  <div class="listening-item">
    <button class="audio-btn" onclick="playAudio(url)">▶ 再生</button>
    <audio id="lis-audio" src="..."></audio>
    
    <div class="audio-script"><!-- スクリプト表示 --></div>
    
    <div class="question">Question: ...</div>
    <div class="choices">
      <label><input type="radio"> (A) ...</label>
      <!-- ... (B)(C)(D) ... -->
    </div>
  </div>
</div>
```

### 2. JavaScript 主要関数

#### `doLogin()` ログイン処理
```javascript
function doLogin() {
  const id = document.getElementById('login-id').value.trim();
  const name = document.getElementById('login-name').value.trim();
  
  // GAS へ POST
  fetch(GAS_URL, {
    method: 'POST',
    body: JSON.stringify({
      action: 'login',
      studentId: id,
      name: name
    })
  })
  .then(r => r.json())
  .then(data => {
    if (data.ok) {
      // ログイン成功 → 問題データ取得
      fetchQuestions();
      goPage('home');
    } else {
      // エラー表示
      document.getElementById('login-error').style.display = 'block';
    }
  });
}
```

**認証フロー**:
1. 学生ID + 氏名を GAS へ送信
2. GAS で スプレッドシート (Sheet1) を照合
3. 一致 → 問題データセット取得
4. 不一致 → エラーメッセージ

#### `fetchQuestions()` 問題データ取得
```javascript
function fetchQuestions() {
  fetch(GAS_URL + '?action=getQuestions&callback=processQuestions')
  .then(...)
  // または
  fetch(GAS_URL, {
    method: 'POST',
    body: JSON.stringify({ action: 'getQuestions' })
  })
  .then(r => r.json())
  .then(data => {
    QD = data.QD;          // 大問1〜4
    LIS_Q = data.LIS_Q;    // リスニング30問
    Q5_MODEL = data.Q5_MODEL;
    Q6_MODEL = data.Q6_MODEL;
    
    // レンダリング
    renderQuestions();
  });
}
```

**返される JSON 構造**:
```javascript
{
  "ok": true,
  "QD": [
    {
      part: 1,
      no: 1,
      question: "問文...",
      choices: ["(A) ...", "(B) ...", "(C) ...", "(D) ..."],
      answer: 2,  // 正答は 0-indexed
      explanation: "解説..."
    },
    // ... 大問1〜4 全問
  ],
  "LIS_Q": [
    {
      part: 1,
      no: 1,
      audioSrc: "https://raw.githubusercontent.com/.../Set2_P1_No1.mp3",
      script: "F: Hi, John. ...\nM: ...",
      question: "Question: What ...",
      choices: ["(A) ...", ...],
      answer: 2,
      explanation: "..."
    },
    // ... リスニング全30問
  ],
  "Q5_MODEL": "モデル解答...",
  "Q5_POINTS": ["✅ 質問1...", ...],
  "Q6_MODEL_YES": "Yes版モデル解答...",
  "Q6_MODEL_NO": "No版モデル解答...",
  "Q6_POINTS": [...]
}
```

#### `renderQuestions()` 問題表示
```javascript
function renderQuestions() {
  // 大問1の問題 HTML を生成
  const q1Html = QD.filter(q => q.part === 1).map(q => `
    <div class="question">
      <div class="q-number">問${q.no}</div>
      <div class="q-text">${q.question}</div>
      <div class="choices">
        ${q.choices.map((c, i) => `
          <label>
            <input type="radio" name="q1-${q.no}" value="${i}" onchange="recordAnswer('q1-${q.no}', ${i})">
            ${c}
          </label>
        `).join('')}
      </div>
    </div>
  `).join('');
  
  document.getElementById('q1-container').innerHTML = q1Html;
}
```

#### `recordAnswer()` 回答記録
```javascript
function recordAnswer(questionId, answerIndex) {
  answers[questionId] = answerIndex;
  
  // localStorageに保存
  localStorage.setItem('answers', JSON.stringify(answers));
}
```

#### `checkSection()` 採点実行
```javascript
function checkSection(section) {
  const sectionQuestions = QD.filter(q => {
    const qNum = parseInt(section[1]);
    return q.part === qNum;
  });
  
  let correct = 0, total = sectionQuestions.length;
  
  sectionQuestions.forEach(q => {
    const userAnswer = answers[`${section}-${q.no}`];
    if (userAnswer === q.answer) correct++;
  });
  
  // 結果表示
  const resultHtml = `
    <div class="result-summary">
      ${section.toUpperCase()} 採点結果: ${correct}/${total} 問 (正答率 ${(correct/total*100).toFixed(1)}%)
    </div>
    <div class="result-details">
      ${sectionQuestions.map(q => {
        const userAnswer = answers[`${section}-${q.no}`];
        const isCorrect = userAnswer === q.answer;
        return `
          <div class="result-item ${isCorrect ? 'correct' : 'incorrect'}">
            <div>問${q.no}: ${isCorrect ? '✓' : '✗'}</div>
            <div>あなたの回答: ${q.choices[userAnswer]}</div>
            <div>正答: ${q.choices[q.answer]}</div>
            <div class="explanation">${q.explanation}</div>
          </div>
        `;
      }).join('')}
    </div>
  `;
  
  document.getElementById(`${section}-result`).innerHTML = resultHtml;
  
  // スコア記録
  scores[section] = { correct, total };
  updateHome();
}
```

#### `submitAnswers()` 送信処理
```javascript
function submitAnswers() {
  const payload = {
    action: 'send',
    studentId: document.getElementById('student-id').value,
    studentName: document.getElementById('student-name').value,
    answers: answers,
    scores: scores,
    timestamp: new Date().toISOString()
  };
  
  fetch(GAS_URL, {
    method: 'POST',
    body: JSON.stringify(payload)
  })
  .then(r => r.json())
  .then(data => {
    if (data.ok) {
      alert('送信完了しました。メールをご確認ください。');
      // ホームへ戻す
      goPage('home');
    }
  });
}
```

### 3. CSS・ デザイン

**カラーパレット**:
```css
:root {
  --bg: #f7f6f3;              /* 背景色 */
  --surface: #fff;            /* 表面色 */
  --accent: #2563c4;          /* アクセント（筆記） */
  --blue-dark: #1a4a8a;       /* リスニング色 */
  --green: #0f6e56;           /* 成功色 */
  --red: #993c1d;             /* エラー色 */
}
```

**レスポンシブ設計**:
```css
/* PC */
.page { max-width: 820px; margin: 0 auto; }

/* モバイル */
@media (max-width: 600px) {
  .page { padding: 1rem 0.75rem; }
  .sec-cards { grid-template-columns: 1fr; }
}
```

---

## ⚙️ バックエンド (Google Apps Script) 実装詳細

### 1. 定数・設定

```javascript
const SHEET_ID = '1wGTBkilJMTZkn-try_KkdENUBudjuU4vXMGO3u-om64';
const SHEET_NAME = 'Sheet1';
const SHEET_RESULTS = '結果集計_Set2';
const MAIL_TO = 'online-eigojuku@toshin.com';
const ACCESS_DEADLINE = new Date('2027-01-25T00:00:00+09:00');
```

### 2. `doGet()` ・ `doPost()`

```javascript
function doGet(e) {
  const params = e.parameter;
  const action = params.action;
  const callback = params.callback || 'callback';
  let result;
  
  try {
    if (action === 'login')        result = handleLogin(params);
    else if (action === 'getQuestions') result = handleGetQuestions(params);
    else if (action === 'send')    result = handleSend(params);
    // ...
  } catch (err) {
    result = { ok: false, error: err.message };
  }
  
  // JSONP で返す
  return ContentService.createTextOutput(
    callback + '(' + JSON.stringify(result) + ')'
  ).setMimeType(ContentService.MimeType.JAVASCRIPT);
}

function doPost(e) {
  let params = JSON.parse(e.postData.contents);
  let result;
  
  try {
    if (params.action === 'login')        result = handleLogin(params);
    else if (params.action === 'getQuestions') result = handleGetQuestions(params);
    // ...
  } catch (err) {
    result = { ok: false, error: err.message };
  }
  
  return ContentService.createTextOutput(
    JSON.stringify(result)
  ).setMimeType(ContentService.MimeType.JSON);
}
```

### 3. `handleLogin()` ログイン認証

```javascript
function handleLogin(params) {
  const inputId = String(params.studentId || '').trim();
  const inputName = String(params.name || '').trim();
  const now = new Date();
  
  // 期限切れチェック（管理者ID除外）
  if (now >= ACCESS_DEADLINE && inputId !== '00000000') {
    return {
      ok: false,
      expired: true,
      message: '利用期間が終了しました'
    };
  }
  
  // スプレッドシートから学生情報を取得
  const sheet = SpreadsheetApp.openById(SHEET_ID).getSheetByName(SHEET_NAME);
  const data = sheet.getDataRange().getValues();
  
  // ヘッダー行をスキップして照合
  for (let i = 1; i < data.length; i++) {
    if (String(data[i][0]).trim() === inputId && 
        String(data[i][1]).trim() === inputName) {
      return {
        ok: true,
        message: 'ログイン成功'
      };
    }
  }
  
  return {
    ok: false,
    message: '生徒番号または氏名が正しくありません'
  };
}
```

**スプレッドシート形式** (Sheet1):
```
列A: 生徒番号    列B: 氏名
00000001       テスト
00000002       田中太郎
00000003       山田花子
```

### 4. `handleGetQuestions()` 問題データ取得

```javascript
function handleGetQuestions(params) {
  // QD (大問1〜4) データ定義
  const QD = [
    {
      part: 1, no: 1,
      question: "( ) 内に当てはまる最も良い語句を選びなさい。",
      choices: ["選択肢A", "選択肢B", "選択肢C", "選択肢D"],
      answer: 2,
      explanation: "これが正答です。理由は..."
    },
    // ... 合計37問 (大問1〜4)
  ];
  
  // LIS_Q (リスニング 30問) データ定義
  const LIS_Q = [
    {
      part: 1, no: 1,
      audioSrc: "https://raw.githubusercontent.com/kj-eiken/eiken-audio/main/Set2_P1_No1.mp3",
      audioLabel: "No. 1 会話を聞く",
      script: "F: ...\nM: ...",
      question: "Question: ...",
      choices: ["...", "...", "...", "..."],
      answer: 2,
      explanation: "..."
    },
    // ... 合計30問 (リスニング全部)
  ];
  
  // 英作文モデル解答
  const Q5_MODEL = "...";
  const Q5_POINTS = ["✅ 質問1...", ...];
  
  const Q6_MODEL_YES = "...";
  const Q6_MODEL_NO = "...";
  const Q6_POINTS = [...];
  
  return {
    ok: true,
    QD: QD,
    LIS_Q: LIS_Q,
    Q5_MODEL: Q5_MODEL,
    Q5_POINTS: Q5_POINTS,
    Q6_MODEL_YES: Q6_MODEL_YES,
    Q6_MODEL_NO: Q6_MODEL_NO,
    Q6_POINTS: Q6_POINTS
  };
}
```

### 5. `handleSend()` メール送信・記録

```javascript
function handleSend(params) {
  const studentId = params.studentId;
  const studentName = params.studentName;
  const scores = params.scores;
  const timestamp = new Date(params.timestamp);
  
  // メール本文を作成
  const mailBody = `
    ご受講ありがとうございます。
    
    【受講者情報】
    生徒番号: ${studentId}
    氏名: ${studentName}
    提出日時: ${timestamp.toLocaleString('ja-JP')}
    
    【採点結果】
    大問1: ${scores.q1.correct}/${scores.q1.total} 問
    大問2: ${scores.q2.correct}/${scores.q2.total} 問
    ...
    
    詳細はアプリ内の結果ページをご確認ください。
  `;
  
  // メール送信
  GmailApp.sendEmail(
    MAIL_TO,
    `[英検準2級 Set2] ${studentId} ${studentName} - 提出通知`,
    mailBody
  );
  
  // スプレッドシートに記録
  const resultsSheet = SpreadsheetApp.openById(SHEET_ID)
                                     .getSheetByName(SHEET_RESULTS);
  
  resultsSheet.appendRow([
    studentId,
    studentName,
    scores.q1.correct,
    scores.q2.correct,
    scores.q3.correct,
    scores.q4.correct,
    // ... その他 
    timestamp.toLocaleString('ja-JP')
  ]);
  
  return {
    ok: true,
    message: 'メール送信・記録完了'
  };
}
```

**スプレッドシート形式** (結果集計_Set2):
```
列A: ID | 列B: 名前 | 列C: 大問1 | ... | 列M: 提出日時
00000001 テスト     12/15              2026/07/01 10:30
```

---

## 🔗 データフロー

### ユーザー操作フロー

```
1. ログイン画面
   ├─ ID + 氏名入力
   └─ 「ログイン」ボタン
      ↓
2. GAS: handleLogin() で認証
   ├─ Sheet1 と照合
   └─ OK → 問題データ取得
      ↓
3. handleGetQuestions() で QD / LIS_Q 取得
   └─ フロントに返す
      ↓
4. ホーム画面表示
   ├─ 大問カード表示
   └─ 進捗バー表示 (0%)
      ↓
5. 大問1をクリック
   ├─ 問題1〜15 表示
   └─ ラジオボタンで選択
      ↓
6. 「解答する」ボタン
   ├─ checkSection('q1') 実行
   ├─ 採点（正答数カウント）
   └─ 結果表示 (例: 12/15 問)
      ↓
7. 全大問+リスニング完了後
   ├─ 「送信」ボタン
   └─ submitAnswers()
      ↓
8. GAS: handleSend()
   ├─ メール送信
   ├─ スプレッドシート記録
   └─ 完了メッセージ
```

---

## 🚨 エラーハンドリング

### フロントエンド
```javascript
// ネットワークエラー
fetch(...).catch(err => {
  console.error('通信エラー:', err);
  alert('通信に失敗しました。もう一度お試しください。');
});

// GAS エラー
.then(data => {
  if (!data.ok) {
    alert(`エラー: ${data.error || data.message}`);
  }
});
```

### バックエンド
```javascript
try {
  // 処理
} catch (err) {
  return {
    ok: false,
    error: err.message,
    stack: err.stack  // デバッグ用
  };
}
```

---

## 🔐 セキュリティ対策

1. **認証**: スプレッドシートを照合（簡易認証）
2. **期限チェック**: 2027年1月25日以後はアクセス不可
3. **セッションタイムアウト**: 15分無操作でログアウト
4. **HTTPS**: GAS URL は自動的に HTTPS
5. **CORS**: GAS の JSONP / POST で対応

---

## 📊 データ仕様

### 問題データ (QD) 仕様
```json
{
  "part": 1,                // 大問番号 (1-4)
  "no": 1,                  // 問番号
  "question": "問文テキスト",
  "choices": [              // 選択肢（4個）
    "(A) 選択肢A",
    "(B) 選択肢B",
    "(C) 選択肢C",
    "(D) 選択肢D"
  ],
  "answer": 2,              // 正答 (0-indexed, 0=A, 1=B, 2=C, 3=D)
  "explanation": "解説テキスト"
}
```

### リスニングデータ (LIS_Q) 仕様
```json
{
  "part": 1,                // リスニング部（1-3）
  "no": 1,                  // 問番号（1-30）
  "audioSrc": "URL",        // 音声ファイルURL
  "audioLabel": "No. 1 ...",
  "script": "F: ...\nM: ...",  // スクリプト
  "question": "Question: ...",
  "choices": [...],
  "answer": 2,
  "explanation": "..."
}
```

---

## 💾 ローカルストレージ仕様

```javascript
// localStorageに保存される回答データ
{
  "answers": {
    "q1-1": 2,    // 大問1 問1: C を選択
    "q1-2": 0,    // 大問1 問2: A を選択
    "q2-1": 1,    // 大問2 問1: B を選択
    ...
  },
  "scores": {
    "q1": { "correct": 12, "total": 15 },
    "q2": { "correct": 4, "total": 5 },
    ...
  }
}
```

---

## 🧪 デバッグ方法

### ブラウザ DevTools
```javascript
// コンソールで実行
console.log(answers);       // 回答データ確認
console.log(scores);        // スコア確認
localStorage.getItem('answers');  // localStorage 確認
```

### GAS ログ確認
```javascript
function testLogin() {
  const result = handleLogin({ studentId: '00000001', name: 'テスト' });
  Logger.log(result);  // Apps Script > ログ で確認
}
```

---

## 📈 今後の拡張案

1. **分析ダッシュボード**: 管理者が学生別・問題別の成績分析
2. **複数ユーザー同時実行**: 同一デバイスで異なる学生がログイン
3. **オフライン対応**: Service Worker で offline 対応
4. **AI採点**: GPT API で英作文を自動採点
5. **スマートフォンアプリ化**: React Native / Flutter

---

**技術ドキュメント版**: 1.0  
**作成日**: 2026年7月1日
