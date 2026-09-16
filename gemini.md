```markdown
# Python部 LXP - giscus直接連携フォーム実装ガイド

Googleフォームの埋め込みを完全廃止し、Webページ上のHTMLフォームから直接giscus（GitHub Discussions）へ投稿する構成の導入手順とコード一式です。

---

## 📐 1. システム構成・フロー

1. **入力**: ユーザーがWebページ上のオリジナルフォームに情報を入力（GitHubログイン不要）。
2. **送信**: JavaScriptの `fetch` APIを用いて、GAS（Google Apps Script）のWeb APIエンドポイントへJSONデータをPOST送信。
3. **連携**: GASが管理者アクセストークン（PAT）を利用してGitHub GraphQL APIを叩き、Discussionを作成。
4. **反映**: giscusが新しいDiscussionを読み込み、右側のタイムラインへ即座に反映。

---

## ⚙️ 2. Google Apps Script (`gas_script.gs`)

Google Apps Scriptエディタに以下のコードを貼り付けます。

```javascript
// ⚙️ 設定項目
const GITHUB_TOKEN = "YOUR_GITHUB_PERSONAL_ACCESS_TOKEN"; // Fine-grained token (Discussions: Read and write)
const REPO_OWNER = "atsukuma81";
const REPO_NAME = "atsukuma81.github.io";
const CATEGORY_ID = "DIC_kwDOTp5gnM4DFax1"; // giscus Announcements Category ID

/**
 * Web APIとしてリクエストを受け取る処理
 */
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    
    const activityDate = data.activityDate || "未設定";
    const userIdentifier = data.userIdentifier || "匿名";
    const title = data.title || "無題の作品";
    const url = data.url || "";
    const description = data.description || "";

    // GitHub Discussion のタイトル
    const discussionTitle = `📌 【活動報告】${title}（${userIdentifier}さん）`;
    
    // Markdown本文の作成
    let body = `### 📅 活動日: ${activityDate}\n`;
    body += `**👤 報告者:** ${userIdentifier}\n`;
    body += `**🎯 作品タイトル:** ${title}\n\n`;
    if (url) {
      body += `**🔗 作品URL:**\n[${url}](${url})\n\n`;
    }
    if (description) {
      body += `**📝 説明・遊び方:**\n${description}\n`;
    }

    // Discussion 作成処理を実行
    const resultUrl = createGitHubDiscussion(discussionTitle, body);

    return ContentService.createTextOutput(JSON.stringify({
      status: "success",
      message: "Discussion created successfully",
      url: resultUrl
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      status: "error",
      message: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

/**
 * GitHub GraphQL API を呼び出して Discussion を作成する
 */
function createGitHubDiscussion(title, body) {
  const graphqlUrl = "[https://api.github.com/graphql](https://api.github.com/graphql)";
  
  // 1. リポジトリIDの取得
  const repoQuery = {
    query: `query {
      repository(owner: "${REPO_OWNER}", name: "${REPO_NAME}") {
        id
      }
    }`
  };

  const repoOptions = {
    method: "post",
    headers: {
      Authorization: `Bearer ${GITHUB_TOKEN}`,
      "Content-Type": "application/json"
    },
    payload: JSON.stringify(repoQuery)
  };

  const repoResponse = UrlFetchApp.fetch(graphqlUrl, repoOptions);
  const repoData = JSON.parse(repoResponse.getContentText());
  const repositoryId = repoData.data.repository.id;

  // 2. Discussion の作成
  const createDiscussionMutation = {
    query: `mutation {
      createDiscussion(input: {
        repositoryId: "${repositoryId}",
        categoryId: "${CATEGORY_ID}",
        title: ${JSON.stringify(title)},
        body: ${JSON.stringify(body)}
      }) {
        discussion {
          url
        }
      }
    }`
  };

  const createOptions = {
    method: "post",
    headers: {
      Authorization: `Bearer ${GITHUB_TOKEN}`,
      "Content-Type": "application/json"
    },
    payload: JSON.stringify(createDiscussionMutation)
  };

  const createResponse = UrlFetchApp.fetch(graphqlUrl, createOptions);
  const createData = JSON.parse(createResponse.getContentText());
  
  return createData.data.createDiscussion.discussion.url;
}

```

---

## 📄 3. 組み込み用 HTML (`index.html`)

`YOUR_GAS_WEB_APP_URL` の部分を発行したGASのウェブアプリURLに書き換えます。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Python部 - 学習共有プラットフォーム (LXP)</title>
    <!-- Tailwind CSS -->
    <script src="[https://cdn.tailwindcss.com](https://cdn.tailwindcss.com)"></script>
    <style>
        .giscus, .giscus-frame {
            width: 100%;
            min-height: 400px;
        }
    </style>
</head>
<body class="bg-gray-50 text-gray-800 min-h-screen">

    <!-- ヘッダー -->
    <header class="bg-blue-600 text-white shadow-md sticky top-0 z-10">
        <div class="max-w-6xl mx-auto px-4 py-4 flex justify-between items-center">
            <h1 class="text-xl font-bold flex items-center gap-2">
                🐍 Python部 LXP
            </h1>
            <span class="text-sm bg-blue-700 px-3 py-1 rounded-full font-medium">活動・作品投稿ボード</span>
        </div>
    </header>

    <!-- メインコンテンツ -->
    <main class="max-w-6xl mx-auto px-4 py-6 grid grid-cols-1 md:grid-cols-12 gap-6">

        <!-- 左側：直接入力フォームエリア (5/12幅) -->
        <div class="md:col-span-5 space-y-4">
            <div class="bg-white p-6 rounded-xl shadow-sm border border-gray-100">
                <h2 class="font-bold text-lg text-gray-900 mb-1 flex items-center gap-2">
                    📝 今日の活動・作品を投稿する
                </h2>
                <p class="text-xs text-gray-500 mb-5">
                    記入して送信すると、右側のタイムライン（giscus）に即時反映されます。
                </p>

                <!-- フォーム本体 -->
                <form id="lxpForm" class="space-y-4">
                    <div>
                        <label for="activityDate" class="block text-xs font-bold text-gray-700 mb-1">活動日 <span class="text-red-500">*</span></label>
                        <input type="date" id="activityDate" name="activityDate" required
                               class="w-full px-3 py-2 text-sm border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500">
                    </div>

                    <div>
                        <label for="userIdentifier" class="block text-xs font-bold text-gray-700 mb-1">名札の番号＆名前orニックネーム <span class="text-red-500">*</span></label>
                        <input type="text" id="userIdentifier" name="userIdentifier" placeholder="例: 05 太郎" required
                               class="w-full px-3 py-2 text-sm border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500">
                    </div>

                    <div>
                        <label for="title" class="block text-xs font-bold text-gray-700 mb-1">作品のタイトル <span class="text-red-500">*</span></label>
                        <input type="text" id="title" name="title" placeholder="例: 数字当てゲーム" required
                               class="w-full px-3 py-2 text-sm border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500">
                    </div>

                    <div>
                        <label for="url" class="block text-xs font-bold text-gray-700 mb-1">作品のURL</label>
                        <input type="url" id="url" name="url" placeholder="[https://trinket.io/](https://trinket.io/)..."
                               class="w-full px-3 py-2 text-sm border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500">
                    </div>

                    <div>
                        <label for="description" class="block text-xs font-bold text-gray-700 mb-1">説明・遊び方</label>
                        <textarea id="description" name="description" rows="4" placeholder="作品の工夫したところや、遊び方を書いてね！"
                                  class="w-full px-3 py-2 text-sm border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"></textarea>
                    </div>

                    <button type="submit" id="submitBtn"
                            class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-2.5 px-4 rounded-md transition duration-200 shadow-sm flex justify-center items-center gap-2">
                        <span>送信して掲示板に反映</span>
                    </button>
                </form>

                <!-- メッセージ表示エリア -->
                <div id="statusMessage" class="hidden mt-4 p-3 rounded-md text-xs text-center font-medium"></div>
            </div>

            <!-- クイックリンク -->
            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="font-bold text-gray-900 mb-2 text-sm">🚀 クイックリンク</h3>
                <ul class="text-sm space-y-2 text-blue-600">
                    <li><a href="[https://prog-8.com](https://prog-8.com)" target="_blank" rel="noopener noreferrer" class="hover:underline">・ Progate (プロゲート) ↗</a></li>
                    <li><a href="[https://trinket.io](https://trinket.io)" target="_blank" rel="noopener noreferrer" class="hover:underline">・ Trinket (Python実行環境) ↗</a></li>
                </ul>
            </div>
        </div>

        <!-- 右側：giscusタイムラインエリア (7/12幅) -->
        <div class="md:col-span-7 space-y-4">
            
            <div class="bg-blue-50 border border-blue-200 p-4 rounded-xl text-sm text-blue-800">
                <p class="font-bold mb-1 flex items-center gap-1">💡 つかいかた</p>
                <ol class="list-decimal list-inside space-y-1 text-xs sm:text-sm">
                    <li>左側のフォームからログイン不要で本日の成果を投稿できます。</li>
                    <li>送信された作品は右側のタイムラインへ自動反映されます。</li>
                    <li>GitHubでログインすると、投稿に対してコメントやリアクション（絵文字）が付けられます！</li>
                </ol>
            </div>

            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h2 class="font-bold text-lg text-gray-900 mb-4 flex items-center gap-2">
                    💬 みんなのタイムライン & 交流広場
                </h2>
                
                <!-- giscus埋め込みエリア -->
                <div id="giscusContainer">
                    <script src="[https://giscus.app/client.js](https://giscus.app/client.js)"
                            data-repo="atsukuma81/atsukuma81.github.io"
                            data-repo-id="R_kgDOTp5gnA"
                            data-category="Announcements"
                            data-category-id="DIC_kwDOTp5gnM4DFax1"
                            data-mapping="pathname"
                            data-strict="0"
                            data-reactions-enabled="1"
                            data-emit-metadata="0"
                            data-input-position="top"
                            data-theme="light"
                            data-lang="ja"
                            crossorigin="anonymous"
                            async>
                    </script>
                </div>
            </div>

        </div>
    </main>

    <footer class="text-center py-6 text-xs text-gray-400">
        &copy; 2026 Python部 LXP Project. All rights reserved.
    </footer>

    <!-- JavaScript：GAS Web API呼び出し処理 -->
    <script>
        // ⚙️ GASのウェブアプリURLをここに貼り付けます
        const GAS_WEB_APP_URL = "YOUR_GAS_WEB_APP_URL";

        // 本日の日付をデフォルト設定
        document.getElementById('activityDate').valueAsDate = new Date();

        const form = document.getElementById('lxpForm');
        const submitBtn = document.getElementById('submitBtn');
        const statusMessage = document.getElementById('statusMessage');

        form.addEventListener('submit', async (e) => {
            e.preventDefault();

            // 送信ボタンの無効化とローディング表示
            submitBtn.disabled = true;
            submitBtn.classList.add('opacity-50', 'cursor-not-allowed');
            submitBtn.innerText = '送信中...';

            statusMessage.classList.add('hidden');

            const formData = {
                activityDate: document.getElementById('activityDate').value,
                userIdentifier: document.getElementById('userIdentifier').value,
                title: document.getElementById('title').value,
                url: document.getElementById('url').value,
                description: document.getElementById('description').value
            };

            try {
                // GASへデータ送信
                const response = await fetch(GAS_WEB_APP_URL, {
                    method: 'POST',
                    headers: { 'Content-Type': 'text/plain' }, // CORS回避用
                    body: JSON.stringify(formData)
                });

                const result = await response.json();

                if (result.status === 'success') {
                    statusMessage.className = "mt-4 p-3 rounded-md text-xs text-center font-medium bg-green-100 text-green-800";
                    statusMessage.innerText = "🎉 投稿が完了しました！タイムラインに反映されるまで数秒お待ちください。";
                    statusMessage.classList.remove('hidden');

                    // フォームのリセット（日付は維持）
                    const currentDate = document.getElementById('activityDate').value;
                    form.reset();
                    document.getElementById('activityDate').value = currentDate;

                    // 3秒後にページ再読み込みしてgiscusを最新化
                    setTimeout(() => {
                        window.location.reload();
                    }, 3000);
                } else {
                    throw new Error(result.message);
                }
            } catch (error) {
                console.error('Submission Error:', error);
                statusMessage.className = "mt-4 p-3 rounded-md text-xs text-center font-medium bg-red-100 text-red-800";
                statusMessage.innerText = "❌ 投稿に失敗しました。もう一度お試しいただくか、管理者へお問い合わせください。";
                statusMessage.classList.remove('hidden');
            } finally {
                submitBtn.disabled = false;
                submitBtn.classList.remove('opacity-50', 'cursor-not-allowed');
                submitBtn.innerText = '送信して掲示板に反映';
            }
        });
    </script>
</body>
</html>

```

---

## 🛠️ デプロイ手順

1. **GASのWeb App化**:
* `gas_script.gs` のコードを貼り付け後、GAS右上「デプロイ」 > 「新しいデプロイ」を選択。
* 種類に **「ウェブアプリ」** を指定。
* アクセスできるユーザーを **「全員」** に指定してデプロイ。
* 発行された **ウェブアプリのURL** を取得。


2. **HTMLへの反映**:
* `index.html` 内の `const GAS_WEB_APP_URL = "YOUR_GAS_WEB_APP_URL";` に上記URLを入力。



```

```
