# 点赞功能 · GitHub 方案配置说明

网站是纯静态页面（GitHub 托管）。要让**别人的点赞也能被记录、并且跨设备共享**，需要一个存放计数的地方。
本方案用 **GitHub Gist** 存计数：

- **读取（显示总数）**：任何访客都能匿名读取 Gist，无需任何密钥 —— 这一步已经在网页里实现好了。
- **写入（+1）**：GitHub 不允许匿名写。静态页面里**绝不能**直接放能写 Gist 的 token（会被所有人看到并盗用），
  所以 +1 要经过一个**极简的免费 serverless 代理**（token 作为服务端密钥，不暴露在前端）。

> 已经实现的部分：点赞后**永久锁定、不可取消**；计数会先读 Gist 全局值，点赞后乐观 +1 并本地永久记住。
> 你只需要按下面两步把「全局写入」接上即可。没接之前，页面也能正常用（每台设备本地永久点赞 + 本地累计）。

---

## 第 1 步：创建存计数的 Gist（1 分钟）

1. 打开 https://gist.github.com/ （用你的 GitHub 账号登录）。
2. 新建一个 **Public**（或 Secret 也行，匿名仍可按 ID 读）Gist：
   - 文件名填：`like-count.json`
   - 内容填：
     ```json
     { "count": 0 }
     ```
3. 创建后，浏览器地址栏里那串十六进制就是 **Gist ID**，形如
   `https://gist.github.com/你的用户名/**a1b2c3d4e5f6...**`，把加粗那段记下来。

然后把 Gist ID 填到 `index.html` 的点赞按钮上（搜索 `id="likeBtn"`）：

```html
<button id="likeBtn" ... data-gist="这里填GistID" data-gist-file="like-count.json" data-like-api="">
```

只做到这一步：网页会**读取并显示** Gist 里的总数（可用来手动改数字做展示），但访客点赞还不会写回云端。

---

## 第 2 步：接一个免费写入代理，让「别人的点赞」真正 +1（约 5 分钟）

推荐用 **Cloudflare Workers**（免费额度足够，无需服务器）。流程：

1. 生成一个 **fine-grained personal access token**：
   GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens →
   仅授予该 Gist 的 **Gists: Read and write** 权限。**这个 token 只放到 Worker 的环境变量里，不要写进网页。**
2. 新建一个 Cloudflare Worker，粘贴下面代码，并在 Worker 的
   **Settings → Variables** 里加两个环境变量：`GH_TOKEN`（上一步的 token）、`GIST_ID`（第 1 步的 Gist ID）。

```js
// Cloudflare Worker：收到 POST 就把 Gist 里的 count +1，返回最新总数
export default {
  async fetch(req, env) {
    const cors = {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type",
    };
    if (req.method === "OPTIONS") return new Response(null, { headers: cors });
    if (req.method !== "POST") return new Response("Method Not Allowed", { status: 405, headers: cors });

    const FILE = "like-count.json";
    const api = `https://api.github.com/gists/${env.GIST_ID}`;
    const gh = {
      "Authorization": `Bearer ${env.GH_TOKEN}`,
      "Accept": "application/vnd.github+json",
      "User-Agent": "like-counter",
    };

    // 读现值
    const cur = await fetch(api, { headers: gh });
    const curJson = await cur.json();
    let count = 0;
    try { count = parseInt(JSON.parse(curJson.files[FILE].content).count, 10) || 0; } catch (_) {}
    count += 1;

    // 写回 +1
    await fetch(api, {
      method: "PATCH",
      headers: { ...gh, "Content-Type": "application/json" },
      body: JSON.stringify({ files: { [FILE]: { content: JSON.stringify({ count }) } } }),
    });

    return new Response(JSON.stringify({ count }), {
      headers: { ...cors, "Content-Type": "application/json" },
    });
  },
};
```

3. 部署后会得到一个地址，形如 `https://xxx.workers.dev`。把它填到点赞按钮的 `data-like-api`：

```html
<button id="likeBtn" ... data-gist="你的GistID" data-gist-file="like-count.json"
        data-like-api="https://xxx.workers.dev">
```

完成。此后任何人在任何设备点赞，都会让 Gist 里的 `count` 累加，所有访客刷新后都能看到同一个总数。

---

## 防刷说明（可选）
- 网页端已做「每台设备只能点一次、且不可取消」。若想更严格（防同一人清缓存反复点），
  可在 Worker 里按 IP 简单去重或加频率限制。个人简历场景通常无需，够用即可。
- 匿名读取 GitHub API 有每小时 60 次/IP 的限制；正常访问量远不会触及。

## 不接后端也能用
- `data-gist` 和 `data-like-api` 都留空时：点赞退化为「本机永久点赞 + 本地累计」，
  功能正常，只是看不到别人的赞。接上上面两步后即变为全局共享。
