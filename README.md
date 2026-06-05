# 历史的回音 (historical-chronological-dialogue-simulator)

> 通过大模型流式生成多位历史学界泰斗或跨时代伟人的第一人称多轮渐进式对话，配合维基百科/百度百科动态肖像与悬浮卡片，以拟真打字机逐字渲染。

![技能类型](https://img.shields.io/badge/技能-WorkBuddy_Skill-blue)
![语言](https://img.shields.io/badge/语言-简体中文-red)

## 效果展示

输入一个问题（如「什么是CPU？」），系统将自动选取 5-8 位代表不同历史时期的关键人物，以编年体递进顺序依次回答，形成从古代到现代的认知演进脉络：

```
帕斯卡 (1642) → 莱布尼茨 (1673) → 巴贝奇 (1837) → 图灵 (1936) → 冯·诺依曼 (1945) → 诺伊斯 (1959) → 法金 (1971)
```

每位人物采用第一人称、口语化对话，保留其历史口癖与人格特质，打字机逐字渲染。

## 核心技术架构

| 维度 | 技术 | 说明 |
|------|------|------|
| LLM 流式生成 | Gemini SDK / SSE | `|||` 分隔符划分人物块，流式输出 |
| 渐进式解析 | 正则前瞻断言 `(?=Name:\s*)` | 非阻塞切割，实时提取 Name/WikiTitle/Era/Text |
| 百科全书检索 | 维基百科→百度百科→快懂百科→搜狗百科 | 四级降级链路，自动获取肖像与简介 |
| 拟真打字机 | 标点自适应停顿 + 随机延时抖动 | 普通字 85-130ms，句末 500-900ms，句中 280-450ms |

## 安装方法

### 方式一：通过 WorkBuddy 技能链接安装（推荐）

在 WorkBuddy 对话中输入：

```
/install-skill https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator
```

### 方式二：手动安装

1. 克隆本仓库到 WorkBuddy 技能目录：

```bash
# 用户级技能（推荐，所有项目可用）
git clone https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator.git \
  ~/.workbuddy/skills/historical-chronological-dialogue-simulator

# 或项目级技能（仅当前项目可用）
git clone https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator.git \
  .workbuddy/skills/historical-chronological-dialogue-simulator
```

2. 重启 WorkBuddy 会话，技能将自动加载

### 方式三：下载 ZIP 安装

1. 点击本页面的 **Code → Download ZIP**
2. 解压到 `~/.workbuddy/skills/historical-chronological-dialogue-simulator/`
3. 重启 WorkBuddy 会话

## 使用方法

在 WorkBuddy 对话中，以下表述将触发本技能：

- "让历史人物讨论原子论"
- "模拟不同时代的学者讨论光是什么"
- "如果古代人聊CPU会怎样"
- `/historical-chronological-dialogue-simulator 什么是物质的量？`

## 环境要求

| 依赖 | 说明 |
|------|------|
| GEMINI_API_KEY | 可选。未配置时自动降级为演示模式（使用预设历史人物数据） |
| 浏览器 | 需支持 `fetch` 和 ES Modules |

## 文件结构

```
historical-chronological-dialogue-simulator/
├── SKILL.md                              # 技能定义（触发词、工作流程、技术栈）
├── README.md                              # 本文件
├── references/
│   ├── prompt-design.md                  # Prompt 模板与编年体约束
│   ├── progressive-parser.md             # 渐进式非阻塞流解析算法
│   ├── wikipedia-integration.md          # 百科全书四级降级集成方案
│   └── typewriter-algorithm.md           # 拟真打字机算法实现
└── assets/
    └── historical-dialogue-template.html # 完整可运行 HTML 模板
```

## 技术细节

### 1. 大模型 Prompt 设计与编年体约束

- 5-8 位历史人物，从最早时代演进到当代
- 严格第一人称、口语化，保留历史口癖
- `|||` 三管道符分隔，配合正则渐进解析
- 强制简体中文输出

### 2. 渐进式非阻塞解析器

流式传输过程中，内容以随机字节块累加。借助前瞻断言正则表达式 `(?=Name:\s*)` 对不完整 Buffer 进行切割，实现边接收边渲染。

### 3. 百科全书四级降级链路

```
维基百科 (WikiTitle+redirects=1)
  → 百度百科 (BaikeLemmaCardApi)
    → 快懂百科 (HTML 抓取)
      → 搜狗百科 (openapi/lemma)
        → 首字取母头像 (最终降级)
```

### 4. 拟真打字机算法

| 字符类型 | 延时范围 | 说明 |
|----------|----------|------|
| 普通字符 | 85-130ms | 正常人跟读速度，约 7-10 字/秒 |
| 句末标点 | 500-900ms | 模拟视线停留与呼吸节奏 |
| 句中停顿 | 280-450ms | 模拟语段间歇 |

## 许可证

MIT License

## 贡献

欢迎提交 Issue 和 Pull Request！
