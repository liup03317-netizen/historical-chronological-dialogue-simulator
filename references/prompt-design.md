# Prompt 设计与编年体约束

## Prompt 模板

```typescript
import { GoogleGenAI } from '@google/genai';

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

const prompt = `You are a group of historical figures having a face-to-face conversation with the user and with each other. 
The user asks: "${question}"
Identify 5 to 8 key historical figures representing major turning points from the earliest times all the way up to the modern era, answering the question step by step. You MUST reach the modern/contemporary understanding by the end of the sequence.
Reply with their chronological answers. 
Make sure their language is colloquial, highly conversational, as if they are sitting in a room with the user, and strictly reflects their historical personas, quirks, and native speaking style.
IMPORTANT: ALL output (Name, Era, Text) MUST be in Simplified Chinese (简体中文).
They should respond to the user and sometimes nod, playfully argue, or reflect upon what the previous speaker just said.

FORMAT:
Separate each figure's response with exactly "|||" (three pipe characters) to avoid formatting conflicts.
For each figure, use this exact format:
Name: [Figure Name in Chinese]
WikiTitle: [Their exact, standard English Wikipedia page title, so we can fetch their portrait]
Era: [Era or Year in Chinese]
Text: [Their explanation/perspective in first person. Conversational tone. Keep it concise, around 3-6 sentences. MUST be in Chinese.]
`;
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

### 输出格式稳定性

- 三管道符 `|||` 作为人物间分隔符
- 每个块必须包含 Name / WikiTitle / Era / Text 四个键值对
- WikiTitle 使用标准英文维基百科标题，确保中文维基能正确重定向
- 全部输出强制简体中文

## 流式调用方式

```typescript
const response = await ai.models.generateContentStream({
  model: 'gemini-2.5-flash',
  contents: prompt,
});

let buffer = '';
for await (const chunk of response) {
  buffer += chunk.text;
  // 将 buffer 传入渐进式解析器
  const figures = parseBuffer(buffer, false);
  // 更新 UI 状态
  updateUI(figures);
}

// 流结束，最终解析
const finalFigures = parseBuffer(buffer, true);
updateUI(finalFigures);
```

## 降级策略

- 若 `GEMINI_API_KEY` 未配置，使用预设历史人物数据（hardcoded）
- 若模型输出格式异常（缺少 `|||` 分隔符），解析器容错降级仍可提取内容
- 若 WikiTitle 为空或维基百科请求失败，使用首字取母头像替代
