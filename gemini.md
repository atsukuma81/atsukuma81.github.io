次のチャットへ引き継ぐための要約と実装用コードのまとめです。そのままコピー＆ペーストしてご利用いただけます。

---

**【前提条件と目的】**

* **目的**: ユーザーがGitHubにログインすることなく「Python部 LXP」のWebページから活動報告を行い、その投稿をgiscus（GitHub Discussions）に自動反映させて、他のユーザーがリアクションや返信をできるようにする。
* **背景**: giscus自体は書き込みにGitHubログインが必須な仕様であるため、GoogleフォームとGoogle Apps Script (GAS) を介した代理投稿構成（構成案2）を採用。
* **運用方法**: フォームは都度作成せず、1つのGoogleフォームを毎週使い回す。

**【システム全体フロー】**

1. ユーザーがWebページ上の**Googleフォーム**から活動報告を送信（ログイン不要）。
2. フォーム送信をトリガーに**GAS**が起動し、管理者のPersonal Access Token (PAT) を使ってGitHub GraphQL APIを叩く。
3. **GitHub Discussions**内に新しいトピックとして自動作成される。
4. Webページ上の**giscus**に投稿が反映され、giscus上で返信やリアクションが可能になる。

**【Googleフォーム側の準備手順】**

1. フォーム項目: 「活動日」「お名前/ニックネーム」「今日の目標・作ったもの」「作品URL」「感想・ひとこと」
2. 設定変更: 「設定」>「回答」>「メールアドレスを収集する: 収集しない」、「回答を1回に制限する: オフ」、「（組織名）のユーザーに限定する: オフ」
3. 画面右上の紫色の「公開」ボタンを押してフォームを公開する。

---

**【コード1: Google Apps Script (GAS) 用コード】**
Googleフォームの「スクリプトエディタ」に貼り付け、フォーム送信時のトリガー設定（`onFormSubmit` / フォームから / フォーム送信時）を行います。

```javascript
// ⚙️ 設定項目
const GITHUB_TOKEN = "YOUR_GITHUB_PERSONAL_ACCESS_TOKEN"; // Fine-grained token (Discussions: Read and write)
const REPO_OWNER = "atsukuma81";
const REPO_NAME = "atsukuma81.github.io";
const CATEGORY_ID = "DIC_kwDOTp5gnM4DFax1"; // giscus Announcements Category ID

function onFormSubmit(e) {
  const itemResponses = e.response.getItemResponses();
  
  let name = "";
  let work = "";
  let url = "";
  let comment = "";

  for (let i = 0; i < itemResponses.length; i++) {
    const title = itemResponses[i].getItem().getTitle();
    const response = itemResponses[i].getResponse();

    if (title.includes("名前")) name = response;
    else if (title.includes("目標") || title.includes("作ったもの")) work = response;
    else if (title.includes("URL")) url = response;
    else if (title.includes("感想") || title.includes("ひとこと")) comment = response;
  }

  const discussionTitle = `📌 【活動報告】${name}さん`;
  
  let body = `### 👤 報告者: ${name}\n`;
  body += `**🎯 今日の目標・作ったもの:**\n${work}\n\n`;
  if (url) body += `**🔗 作成したプログラム:**\n[${url}](${url})\n\n`;
  if (comment) body += `**💬 ひとこと・感想:**\n${comment}\n`;

  createGitHubDiscussion(discussionTitle, body);
}

function createGitHubDiscussion(title, body) {
  const graphqlUrl = "https://api.github.com/graphql";
  
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

  UrlFetchApp.fetch(graphqlUrl, createOptions);
}

```

---

**【コード2: 組み込み用 HTML コード】**
`YOUR_FORM_ID` の部分を発行したGoogleフォームの埋め込みURLに差し替えて使用します。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Python部 - 学習共有プラットフォーム (LXP)</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        .giscus, .giscus-frame {
            width: 100%;
            min-height: 380px;
        }
    </style>
</head>
<body class="bg-gray-50 text-gray-800 min-h-screen">

    <!-- ヘッダー -->
    <header class="bg-blue-600 text-white shadow-md sticky top-0 z-10">
        <div class="max-w-5xl mx-auto px-4 py-4 flex justify-between items-center">
            <h1 class="text-xl font-bold flex items-center gap-2">
                🐍 Python部 LXP
            </h1>
            <span class="text-sm bg-blue-700 px-3 py-1 rounded-full font-medium">みんなの活動共有ボード</span>
        </div>
    </header>

    <!-- メインコンテンツ -->
    <main class="max-w-5xl mx-auto px-4 py-6 grid grid-cols-1 md:grid-cols-12 gap-6">

        <!-- 左側：Googleフォーム埋め込みエリア (5/12幅) -->
        <div class="md:col-span-5 space-y-4">
            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h2 class="font-bold text-lg text-gray-900 mb-2 flex items-center gap-2">
                    📝 今日の活動を報告する
                </h2>
                <p class="text-xs text-gray-500 mb-4">
                    フォームに入力して送信すると、右側のタイムライン（giscus）に自動で反映されます！
                </p>

                <!-- Googleフォームのiframe -->
                <iframe src="https://docs.google.com/forms/d/e/YOUR_FORM_ID/viewform?embedded=true" 
                        width="100%" 
                        height="650" 
                        frameborder="0" 
                        marginheight="0" 
                        marginwidth="0"
                        class="rounded-lg">読み込んでいます…</iframe>
            </div>

            <!-- クイックリンク -->
            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="font-bold text-gray-900 mb-2 text-sm">🚀 クイックリンク</h3>
                <ul class="text-sm space-y-2 text-blue-600">
                    <li><a href="https://prog-8.com" target="_blank" rel="noopener noreferrer" class="hover:underline">・ Progate (プロゲート) ↗</a></li>
                    <li><a href="https://trinket.io" target="_blank" rel="noopener noreferrer" class="hover:underline">・ Trinket (Python実行環境) ↗</a></li>
                </ul>
            </div>
        </div>

        <!-- 右側：giscusタイムラインエリア (7/12幅) -->
        <div class="md:col-span-7 space-y-4">
            
            <div class="bg-blue-50 border border-blue-200 p-4 rounded-xl text-sm text-blue-800">
                <p class="font-bold mb-1 flex items-center gap-1">💡 つかいかた</p>
                <ol class="list-decimal list-inside space-y-1 text-xs sm:text-sm">
                    <li>左側のフォームからログイン不要で今日の活動を報告できます。</li>
                    <li>投稿された報告は右側のタイムラインに自動掲載されます。</li>
                    <li>GitHubでサインインすると、投稿に対してコメントやリアクション（絵文字）ができます！</li>
                </ol>
            </div>

            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h2 class="font-bold text-lg text-gray-900 mb-4 flex items-center gap-2">
                    💬 みんなのタイムライン & 交流広場
                </h2>
                
                <!-- giscus埋め込みエリア -->
                <div id="giscusContainer">
                    <script src="https://giscus.app/client.js"
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

</body>
</html>

```
