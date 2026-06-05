# 渐进式非阻塞流解析器

## 核心问题

大模型采用流式传输，内容以随机字节块不断累加。如果等到全部生成完毕再渲染，用户会经历漫长白屏。渐进式解析器在流式传输过程中实时提取结构化数据，实现边接收边渲染。

## 数据类型定义

```typescript
export interface HistoricalFigureResponse {
  name: string;
  wikiTitle: string;
  era: string;
  text: string;
  isComplete: boolean;
}
```

## 解析算法

```typescript
function parseBuffer(buffer: string, isFinished: boolean): HistoricalFigureResponse[] {
  // 1. 兼容模型偶尔出现的分隔符变形
  const cleanBuffer = buffer.replace(/\|\|\|/g, '\n').replace(/---/g, '\n');
  
  // 2. 利用前瞻断言前缀 "Name:" 完成数据块高容错分割
  const blocks = cleanBuffer.split(/(?=Name:\s*)/i).map(b => b.trim()).filter(b => b.length > 0);

  const figures: HistoricalFigureResponse[] = [];

  for (let i = 0; i < blocks.length; i++) {
    const block = blocks[i];
    const nameMatch = block.match(/Name:\s*(.+)/i);
    const wikiMatch = block.match(/WikiTitle:\s*(.+)/i);
    const eraMatch = block.match(/Era:\s*(.+)/i);
    const textMatch = block.match(/Text:\s*([\s\S]*)/i);
    
    // 如果不是最后一个块，或者整个流已届末尾，则标记为已完成
    const isComplete = i < blocks.length - 1 || isFinished;

    if (nameMatch || eraMatch || textMatch) {
      let text = textMatch ? textMatch[1].trim() : '';

      // 容错降级：如果模型没有正确输出 Text: 前缀，则过滤掉 Name / Era 等行后，自动拼接剩余文本
      if (!text) {
        const lines = block.split('\n').map(l => l.trim()).filter(l => l.length > 0);
        const textLines = lines.filter(l => 
          !l.toLowerCase().startsWith('name:') && 
          !l.toLowerCase().startsWith('era:') && 
          !l.toLowerCase().startsWith('wikititle:')
        );
        text = textLines.join('\n');
      }

      // 清除可能渗入的干扰字符
      text = text.replace(/\|\|\|/g, '').trim();

      figures.push({
        name: nameMatch ? nameMatch[1].trim() : '',
        wikiTitle: wikiMatch ? wikiMatch[1].trim() : '',
        era: eraMatch ? eraMatch[1].trim() : '',
        text: text,
        isComplete
      });
    }
  }
  return figures;
}
```

## 关键设计决策

### 为什么使用前瞻断言 `(?=Name:\s*)`？

普通 `split` 会消费掉分隔符，导致分割后的块丢失 `Name:` 前缀。前瞻断言 `(?=...)` 只匹配位置不消费字符，因此每个块都完整保留了 `Name:` 前缀。

### 为什么标记 `isComplete`？

流式传输中，最后一个块可能仍在生成中，其 `text` 字段不完整。只有当出现下一个 `Name:` 标记或流已结束时，才能确认前一个块完整。`isComplete` 用于驱动打字机动画是否继续。

### 分隔符变形容错

模型可能输出 `---` 或其他变体代替 `|||`，解析器统一替换为换行符处理。

## 集成模式

```typescript
// 在流式接收循环中
let buffer = '';
for await (const chunk of stream) {
  buffer += chunk.text;
  const figures = parseBuffer(buffer, false);
  setFigures(figures); // 更新 React 状态
}

// 流结束
const finalFigures = parseBuffer(buffer, true);
setFigures(finalFigures);
```
