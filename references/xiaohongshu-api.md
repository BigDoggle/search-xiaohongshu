# 小红书接口参考

## 目录

1. 已核验范围
2. 搜索接口
3. 搜索表达式
4. 结果字段
5. 正文请求
6. 登录和错误处理
7. 前端升级后的重新定位

## 1. 已核验范围

核验日期：2026-07-29。

当前网页：

- 页面来源：`https://www.xiaohongshu.com`
- 搜索页面：`/search_result_ai`
- 搜索接口路径：`/api/sns/web/v2/search/notes`
- 页面请求模块会将接口发送到小红书官方搜索域，并自动处理登录态、签名和字段转换
- 笔记详情最终路径：`/explore/{noteId}`

不要把当前 JavaScript 文件哈希、Webpack 模块编号或签名头写死。前端部署后这些值可能变化。

## 2. 搜索接口

页面内部使用的业务参数为：

```javascript
{
  keyword: "关键词",
  page: 1,
  pageSize: 20,
  searchId: "由页面 createSearchId 生成",
  sort: "general",
  noteType: 0,
  extFlags: [],
  filters: [],
  geo: "",
  imageFormats: ["jpg", "webp", "avif"],
  messageId: ""
}
```

页面请求模块会把字段转换成接口实际使用的下划线形式，并生成动态请求头。不要自行拼装 `X-s`、`X-t` 或 `X-S-Common`，也不要从网络事件中复制这些值。

排序值：

```text
general                 综合
time_descending         最新
popularity_descending   最热
```

笔记类型：

```text
0  全部
1  视频
2  图文
```

## 3. 搜索表达式

在 Node 侧先构造并安全编码输入：

```javascript
const searchInput = {
  keyword,
  sort,
  noteType,
  limit: Math.min(Math.max(Number(limit || 10), 1), 40)
};

const inputLiteral = JSON.stringify(searchInput).replaceAll("<", "\\u003c");
```

在小红书标签的 CDP `Runtime.evaluate` 中执行以下固定表达式。用户输入只能通过 `INPUT_LITERAL` 进入：

```javascript
(async input => {
  if (location.origin !== "https://www.xiaohongshu.com") {
    throw new Error("当前标签不是小红书页面");
  }

  const allowedSorts = new Set([
    "general",
    "time_descending",
    "popularity_descending"
  ]);
  const sort = allowedSorts.has(input.sort) ? input.sort : "general";
  const noteType = [0, 1, 2].includes(Number(input.noteType))
    ? Number(input.noteType)
    : 0;
  const limit = Math.min(Math.max(Number(input.limit || 10), 1), 40);

  const chunks = window.webpackChunkxhs_pc_web;
  if (!Array.isArray(chunks)) {
    throw new Error("小红书页面主模块尚未加载");
  }

  let webpackRequire = null;
  chunks.push([
    [Date.now() % 1000000000],
    {},
    runtime => {
      webpackRequire = runtime;
    }
  ]);
  if (!webpackRequire?.m) {
    throw new Error("未取得小红书页面模块加载器");
  }

  let searchFunction = null;
  let createSearchId = null;

  for (const [moduleId, factory] of Object.entries(webpackRequire.m)) {
    const source = String(factory);

    if (!searchFunction && source.includes("WEB_AI_SEARCH_NOTES_V2")) {
      const exports = webpackRequire(moduleId);
      searchFunction = Object.values(exports).find(value =>
        typeof value === "function" &&
        (
          value.name === "getAiSearchNotesV2" ||
          String(value).includes("WEB_AI_SEARCH_NOTES_V2")
        )
      ) || null;
    }

    if (!createSearchId && source.includes("createSearchId")) {
      const exports = webpackRequire(moduleId);
      createSearchId = Object.values(exports).find(value =>
        typeof value === "function" && value.name === "createSearchId"
      ) || null;
    }

    if (searchFunction && createSearchId) break;
  }

  if (!searchFunction || !createSearchId) {
    throw new Error("小红书搜索模块已变化，需要重新探索");
  }

  const searchId = createSearchId();
  const pages = Math.min(Math.ceil(limit / 20), 2);
  const collected = [];
  let hasMore = true;

  for (let page = 1; page <= pages && hasMore; page += 1) {
    const result = await searchFunction({
      keyword: String(input.keyword || "").trim(),
      page,
      pageSize: 20,
      searchId,
      sort,
      noteType,
      extFlags: [],
      filters: [],
      geo: "",
      imageFormats: ["jpg", "webp", "avif"],
      messageId: ""
    });

    const items = Array.isArray(result?.items) ? result.items : [];
    hasMore = Boolean(result?.hasMore);

    for (const item of items) {
      const card = item?.noteCard;
      if (!item?.id || !card || !item?.xsecToken) continue;

      const url = new URL(
        `/explore/${encodeURIComponent(item.id)}`,
        "https://www.xiaohongshu.com"
      );
      url.searchParams.set("xsec_token", item.xsecToken);
      url.searchParams.set("xsec_source", "pc_search");

      const interact = card.interactInfo || {};
      const user = card.user || {};
      const cover = card.cover || {};
      const coverUrl = cover.urlDefault || cover.urlPre || "";
      const imageUrls = Array.isArray(card.imageList)
        ? card.imageList
            .map((image) => {
              const infoList = Array.isArray(image?.infoList)
                ? image.infoList
                : [];
              return (
                infoList.find((info) => info?.imageScene === "WB_DFT")?.url ||
                infoList.find((info) => typeof info?.url === "string")?.url ||
                ""
              );
            })
            .filter(Boolean)
            .slice(0, 18)
        : [];

      if (imageUrls.length === 0 && coverUrl) {
        imageUrls.push(coverUrl);
      }

      collected.push({
        id: item.id,
        title: card.displayTitle || "",
        author: user.nickname || user.nickName || "",
        authorId: user.userId || "",
        noteType: card.type || "",
        likedCount: interact.likedCount || "0",
        collectedCount: interact.collectedCount || "0",
        commentCount: interact.commentCount || "0",
        sharedCount: interact.sharedCount || "0",
        coverUrl,
        imageUrls,
        url: url.href
      });
    }
  }

  const unique = [];
  const seen = new Set();
  for (const item of collected) {
    if (seen.has(item.id)) continue;
    seen.add(item.id);
    unique.push(item);
    if (unique.length >= limit) break;
  }

  return {
    keyword: String(input.keyword || "").trim(),
    sort,
    noteType,
    count: unique.length,
    hasMore,
    results: unique
  };
})(INPUT_LITERAL)
```

调用方式：

```javascript
const cdp = await tab.capabilities.get("cdp");
const response = await cdp.send("Runtime.evaluate", {
  expression: SEARCH_EXPRESSION,
  awaitPromise: true,
  returnByValue: true
}, { timeoutMs: 20000 });
```

读取 `response.result.value`。若存在 `exceptionDetails`，只向用户概括错误，不返回页面堆栈、签名值或完整请求配置。

## 4. 结果字段

搜索模块返回的主要结构：

```text
hasMore
items[]
  id
  xsecToken
  noteCard
    displayTitle
    type
    user
      nickname / nickName
      userId
    interactInfo
      likedCount
      collectedCount
      commentCount
      sharedCount
    cover
      urlDefault
      urlPre
    imageList
      infoList
        imageScene
        url
```

部分结果可能是热词、推荐或其他模型类型，没有标准笔记卡片。只保留同时具有 `id`、`noteCard` 和访问参数的项目。

互动量可能是字符串或缩写，不要未经核验强制转成整数。

## 5. 正文请求

正文输入必须来自本次搜索结果，或经过严格来源和路径校验的用户链接。

Node 侧：

```javascript
const detailInput = {
  urls: urls.slice(0, 5),
  maxLength: 8000
};
const detailLiteral = JSON.stringify(detailInput).replaceAll("<", "\\u003c");
```

页面表达式：

```javascript
(async input => {
  const maxLength = Math.min(
    Math.max(Number(input.maxLength || 8000), 500),
    12000
  );
  const urls = Array.isArray(input.urls) ? input.urls.slice(0, 5) : [];
  const results = new Array(urls.length);

  async function readNote(rawUrl) {
    const url = new URL(rawUrl, "https://www.xiaohongshu.com");
    if (
      url.origin !== "https://www.xiaohongshu.com" ||
      !/^\/(explore|search_result)\/[0-9a-f]+$/i.test(url.pathname)
    ) {
      return { url: rawUrl, error: "链接不属于允许的小红书笔记路径" };
    }

    try {
      const response = await fetch(url.href, {
        credentials: "same-origin",
        redirect: "follow"
      });
      const html = await response.text();
      const doc = new DOMParser().parseFromString(html, "text/html");

      if (
        response.status === 401 ||
        response.status === 403 ||
        new URL(response.url).pathname.includes("/login")
      ) {
        return { url: rawUrl, error: "小红书登录状态不可用" };
      }

      const title = (
        doc.querySelector("#detail-title")?.textContent ||
        ""
      ).trim();
      const content = (
        doc.querySelector("#detail-desc")?.textContent ||
        ""
      ).trim();
      const author = (
        doc.querySelector(".author-container .username")?.textContent ||
        ""
      ).trim();
      const publishedAt = (
        doc.querySelector(".bottom-container")?.textContent ||
        ""
      ).trim();
      const topics = Array.from(
        doc.querySelectorAll('#detail-desc a[href*="/search_result?keyword="]')
      ).map(element => (element.textContent || "").trim()).filter(Boolean);

      return {
        url: response.url,
        title,
        untitled: title.length === 0,
        author,
        publishedAt,
        content: content.slice(0, maxLength),
        truncated: content.length > maxLength,
        topics: topics.slice(0, 30)
      };
    } catch {
      return {
        url: rawUrl,
        error: "笔记正文读取失败"
      };
    }
  }

  let nextIndex = 0;

  async function worker() {
    while (nextIndex < urls.length) {
      const index = nextIndex;
      nextIndex += 1;
      results[index] = await readNote(urls[index]);
    }
  }

  const workerCount = Math.min(2, urls.length);
  await Promise.all(Array.from({ length: workerCount }, () => worker()));

  return results;
})(DETAIL_INPUT_LITERAL)
```

不要返回 `html`、全部脚本、评论区正文或响应头。若标题和正文都为空，视为页面结构变化或访问受限。

## 6. 登录和错误处理

常见情况：

- `401/403`：登录失效或权限不足；
- 出现登录弹窗：让用户在代理标签登录；
- 出现安全验证或验证码：停止请求并交给用户；
- 搜索模块抛出“网络连接不可用”：页面安全模块未加载完成，可等待后重试一次；
- 返回空结果但没有错误：正常报告没有匹配内容；
- `Runtime.evaluate` 超时：只读请求可以重试一次，之后降低结果数量。

不要输出 Axios 配置、请求头、Cookie、签名或浏览器存储。

## 7. 前端升级后的重新定位

模块查找失败时：

1. 在代理标签打开一次参数化搜索页；
2. 启用 CDP 网络事件；
3. 观察 `Network.requestWillBeSent` 和 `Network.responseReceived`；
4. 只记录官方域名、方法、路径、请求字段名和响应字段名；
5. 在当前主脚本中搜索搜索接口路径和 `getAiSearchNotesV2` 附近代码；
6. 更新动态模块识别标记；
7. 用一个只读关键词重新进行冒烟测试。

不要记录 Cookie、完整请求头、动态签名值、完整响应或用户个人信息。
