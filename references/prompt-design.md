# Prompt 设计与编年体约束

## 设计原则

**本技能由 agent 自身生成对话内容，无需调用任何外部 LLM API。** Agent 按照以下 Prompt 约束直接生成编年体对话。

## Prompt 模板

当用户提出问题后，agent 按以下逻辑生成对话内容：

```
用户提问："${question}"

请按编年体顺序，选取5-8位代表关键历史转折点的先贤，以第一人称、口语化方式回答该问题。

约束：
1. 人物必须按时间顺序排列，从最早的时代一直到现代
2. 必须覆盖问题所涉领域的重大历史转折点
3. 最终一位人物必须代表当代/现代的理解水平
4. 严格采用第一人称（"我认为..."、"在我的时代..."）
5. 口语化、对话感，仿佛与用户面对面交谈
6. 保留历史人物的标志性口癖和思维特质
7. 人物之间应有互动：点头认可、幽默争论、对前人观点的反思
8. 全部输出强制简体中文
9. 每位人物的对话保持3-6句

输出格式（每个人物）：
Name: [中文名]
WikiTitle: [英文维基百科标准标题]
Era: [时代或年份，中文]
Text: [第一人称对话，简体中文]
```

## Agent 输出格式

Agent 生成对话后，将内容格式化为以下 JS 对象数组，注入 HTML 模板：

```javascript
[
  {
    name: '德谟克利特',
    wikiTitle: 'Democritus',
    era: '约公元前460年',
    text: '朋友，在我看来，世间万物都是由不可分割的微小粒子组成的...',
    isComplete: true
  },
  {
    name: '约翰·道尔顿',
    wikiTitle: 'John Dalton',
    era: '1803年',
    text: '德谟克利特老前辈说得太对了，不过我得补充一些更精确的东西...',
    isComplete: true
  }
  // ... 更多人物
]
```

## 关键约束说明

### 编年体递进约束

- 人物必须按时间顺序排列，从最早的时代一直到现代
- 必须覆盖用户问题所涉领域的重大历史转折点
- 最终一位人物必须代表当代/现代的理解水平

### 人设注入规则

- 严格采用第一人称（"我认为..."、"在我的时代..."）
- 口语化、对话感，仿佛与用户面对面交谈
- 保留历史人物的标志性口癖和思维特质（如苏格拉底的追问法、牛顿的力学类比）
- 人物之间应有互动：点头认可、幽默争论、对前人观点的反思

### WikiTitle 规则

- WikiTitle 使用标准英文维基百科标题，确保中文维基能正确重定向
- 例如：`Democritus`、`John Dalton`、`Albert Einstein`
- 如果不确定精确标题，使用最通用的英文全名
- 全部输出强制简体中文

## 降级策略

- **不依赖任何外部 API**：Agent 自身即为 LLM，直接生成内容
- 若模型输出格式异常（缺少分隔符），解析器容错降级仍可提取内容
- 若 WikiTitle 为空或维基百科请求失败，使用首字取母头像替代

## 渐进式流解析器（可选高级用法）

如果 agent 支持流式输出，可使用 `|||` 分隔符 + `parseBuffer()` 实现边生成边渲染：

```typescript
// 流式解析器使用示例
let buffer = '';
for await (const chunk of streamOutput) {
  buffer += chunk.text;
  const figures = parseBuffer(buffer, false);
  updateUI(figures);
}
const finalFigures = parseBuffer(buffer, true);
```

**完整解析算法** 详见 `references/progressive-parser.md`。
