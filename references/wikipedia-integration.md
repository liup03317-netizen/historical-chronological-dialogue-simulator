# 百科全书跨语种媒体解析

## 降级链路

按以下优先级依次尝试百科数据源，任一成功即返回：

| 优先级 | 数据源 | 搜索键 | CORS 支持 | 说明 |
|--------|--------|--------|-----------|------|
| 1 | 中文维基百科 | 英文 WikiTitle（自动重定向至中文） | ✅ `origin=*` | 首选，跨语种重定向优雅 |
| 2 | 百度百科 | 中文名称（Name 字段） | ❌ 需代理/后端 | 中文百科覆盖率最高 |
| 3 | 快懂百科 | 中文名称（Name 字段） | ❌ 需代理/后端 | 字节跳动旗下，内容较新 |
| 4 | 搜狗百科 | 中文名称（Name 字段） | ❌ 需代理/后端 | 补充来源 |

## 第一优先：维基百科

直接通过英文 WikiTitle 请求中文维基百科，利用 `redirects=1` 参数自动处理跨语种重定向，一次性获取肖像图和中文简介。

### API 端点

```
https://zh.wikipedia.org/w/api.php?action=query&titles={WikiTitle}&prop=pageimages|extracts&exsentences=2&exintro=true&explaintext=true&format=json&pithumbsize=200&origin=*&redirects=1
```

### 参数说明

| 参数 | 值 | 说明 |
|------|------|------|
| `action` | `query` | 查询操作 |
| `titles` | 英文 WikiTitle | 如 `John Dalton`、`Albert Einstein` |
| `prop` | `pageimages\|extracts` | 同时请求图片和摘要 |
| `exsentences` | `2` | 摘要限制为 2 句 |
| `exintro` | `true` | 仅取引言部分 |
| `explaintext` | `true` | 纯文本格式 |
| `format` | `json` | JSON 返回 |
| `pithumbsize` | `200` | 缩略图 200px |
| `origin` | `*` | 跨域请求支持 |
| `redirects` | `1` | 自动跟踪重定向 |

### 重定向工作原理

1. 大模型生成标准英文 WikiTitle，如 `John Dalton`
2. 中文维基百科收到请求后，通过 `redirects=1` 自动查找对应的中文页面「约翰·道尔顿」
3. 返回的 JSON 中 `query.redirects` 数组记录了重定向路径
4. 肖像图和摘要直接从重定向后的中文页面提取

### 返回数据结构示例

```json
{
  "query": {
    "redirects": [
      { "from": "John Dalton", "to": "约翰·道尔顿" }
    ],
    "pages": {
      "12345": {
        "pageid": 12345,
        "ns": 0,
        "title": "约翰·道尔顿",
        "thumbnail": {
          "source": "https://upload.wikimedia.org/.../John_Dalton_by_Charles_Turner.jpg/200px-John_Dalton_by_Charles_Turner.jpg",
          "width": 200,
          "height": 240
        },
        "extract": "约翰·道尔顿，英国化学家、物理学家，近代原子理论的提出者。..."
      }
    }
  }
}
```

## 第二优先：百度百科

维基百科不可达或未找到对应条目时，使用百度百科开放 API 查询。

### API 端点

```
https://baike.baidu.com/api/openapi/BaikeLemmaCardApi?scope=103&format=json&appid=379020&bk_key={encodeURIComponent(chineseName)}&length=2
```

### 参数说明

| 参数 | 值 | 说明 |
|------|------|------|
| `scope` | `103` | 查询范围（词条摘要+图片） |
| `format` | `json` | JSON 返回 |
| `appid` | `379020` | 固定应用 ID |
| `bk_key` | 中文名称 | 如 `约翰·道尔顿`、`阿尔伯特·爱因斯坦` |
| `length` | `2` | 摘要长度（短摘要） |

### 返回数据结构示例

```json
{
  "key": "约翰·道尔顿",
  "abstract": "约翰·道尔顿（John Dalton，1766年9月6日－1844年7月27日），英国化学家、物理学家。",
  "image": "https://bkimg.cdn.bcebos.com/pics/...",
  "title": "约翰·道尔顿",
  "lemmaId": "12345"
}
```

### 数据提取

- **头像**：`response.image` 字段
- **简介**：`response.abstract` 字段

### CORS 限制

百度百科 API 不支持 CORS。在纯前端环境下需通过代理服务访问；在 WorkBuddy 环境下可使用 `WebFetch` 工具绕过 CORS 限制。

## 第三优先：快懂百科

百度百科也未命中时，尝试快懂百科（字节跳动旗下）。

### 查询方式

快懂百科无公开 JSON API，需通过页面抓取获取：

```
https://www.baike.com/wiki/{encodeURIComponent(chineseName)}
```

### 数据提取

从 HTML 页面中提取：
- **头像**：`<meta property="og:image">` 或结构化数据中的图片 URL
- **简介**：`<meta name="description">` 或词条摘要区域

### 实现建议

在 WorkBuddy 环境中使用 `WebFetch` 工具抓取页面并提取数据：

```typescript
// 伪代码：通过 WebFetch 工具获取快懂百科数据
const html = await webFetch(`https://www.baike.com/wiki/${chineseName}`);
const avatarMatch = html.match(/<meta\s+property="og:image"\s+content="([^"]+)"/);
const descMatch = html.match(/<meta\s+name="description"\s+content="([^"]+)"/);
```

## 第四优先：搜狗百科

作为最终补充来源。

### 查询方式

搜狗百科有公开搜索 API：

```
https://baike.sogou.com/api/openapi/lemma?title={encodeURIComponent(chineseName)}&length=2
```

### 备选：页面抓取

```
https://baike.sogou.com/v{lemmaId}.htm
```

### 数据提取

- **头像**：API 返回的 `imageUrl` 字段或 HTML 中的 `<meta property="og:image">`
- **简介**：API 返回的 `abstract` 字段

## 统一降级函数

```typescript
interface FigureMetadata {
  avatar: string | null;
  bio: string | null;
}

const metadataCache = {};

async function fetchFigureMetadata(wikiTitle: string, chineseName: string): Promise<FigureMetadata> {
  const cacheKey = wikiTitle || chineseName;
  if (metadataCache[cacheKey]) return metadataCache[cacheKey];

  let result: FigureMetadata = { avatar: null, bio: null };

  // 第一优先：维基百科
  result = await fetchFromWikipedia(wikiTitle);
  if (result.avatar || result.bio) {
    metadataCache[cacheKey] = result;
    return result;
  }

  // 第二优先：百度百科
  result = await fetchFromBaiduBaike(chineseName);
  if (result.avatar || result.bio) {
    metadataCache[cacheKey] = result;
    return result;
  }

  // 第三优先：快懂百科（需 WebFetch 工具或代理）
  result = await fetchFromKuaidongBaike(chineseName);
  if (result.avatar || result.bio) {
    metadataCache[cacheKey] = result;
    return result;
  }

  // 第四优先：搜狗百科
  result = await fetchFromSogouBaike(chineseName);
  if (result.avatar || result.bio) {
    metadataCache[cacheKey] = result;
    return result;
  }

  // 全部失败：使用首字取母头像，不显示悬浮卡片
  metadataCache[cacheKey] = { avatar: null, bio: null };
  return { avatar: null, bio: null };
}

// ============================================================
// 各数据源实现
// ============================================================

async function fetchFromWikipedia(wikiTitle: string): Promise<FigureMetadata> {
  if (!wikiTitle) return { avatar: null, bio: null };
  const url = `https://zh.wikipedia.org/w/api.php?action=query&titles=${encodeURIComponent(wikiTitle)}&prop=pageimages|extracts&exsentences=2&exintro=true&explaintext=true&format=json&pithumbsize=200&origin=*&redirects=1`;

  try {
    const response = await fetch(url);
    const data = await response.json();
    const pages = data.query?.pages;
    if (pages) {
      const pageId = Object.keys(pages)[0];
      if (pageId !== '-1') {
        return {
          avatar: pages[pageId].thumbnail?.source || null,
          bio: pages[pageId].extract || null
        };
      }
    }
  } catch (error) {
    console.warn(`Wikipedia fetch failed for ${wikiTitle}:`, error);
  }
  return { avatar: null, bio: null };
}

async function fetchFromBaiduBaike(chineseName: string): Promise<FigureMetadata> {
  if (!chineseName) return { avatar: null, bio: null };
  const url = `https://baike.baidu.com/api/openapi/BaikeLemmaCardApi?scope=103&format=json&appid=379020&bk_key=${encodeURIComponent(chineseName)}&length=2`;

  try {
    const response = await fetch(url);
    if (!response.ok) return { avatar: null, bio: null };
    const data = await response.json();
    if (data.lemmaId || data.title) {
      return {
        avatar: data.image || null,
        bio: data.abstract || null
      };
    }
  } catch (error) {
    console.warn(`Baidu Baike fetch failed for ${chineseName}:`, error);
  }
  return { avatar: null, bio: null };
}

async function fetchFromKuaidongBaike(chineseName: string): Promise<FigureMetadata> {
  // 快懂百科无公开 JSON API，需通过 HTML 抓取
  // 纯前端环境受 CORS 限制，此函数在 WorkBuddy 环境中使用 WebFetch 工具实现
  // 前端 HTML 模板中跳过此步骤
  return { avatar: null, bio: null };
}

async function fetchFromSogouBaike(chineseName: string): Promise<FigureMetadata> {
  if (!chineseName) return { avatar: null, bio: null };
  const url = `https://baike.sogou.com/api/openapi/lemma?title=${encodeURIComponent(chineseName)}&length=2`;

  try {
    const response = await fetch(url);
    if (!response.ok) return { avatar: null, bio: null };
    const data = await response.json();
    if (data.lemmaId || data.title) {
      return {
        avatar: data.imageUrl || data.image || null,
        bio: data.abstract || data.description || null
      };
    }
  } catch (error) {
    console.warn(`Sogou Baike fetch failed for ${chineseName}:`, error);
  }
  return { avatar: null, bio: null };
}
```

## 降级策略总览

- **维基百科失败**（网络不通、页面不存在、CORS 受限）→ 尝试百度百科
- **百度百科失败**（未收录、API 错误）→ 尝试快懂百科
- **快懂百科失败**（无公开 API、CORS 受限）→ 尝试搜狗百科
- **全部失败**：使用首字取母头像，不显示悬浮卡片
- 可并发请求多位人物的元数据，使用 `Promise.all` 或 `Promise.allSettled`
- 每个数据源结果均缓存至 `metadataCache`，避免重复请求
