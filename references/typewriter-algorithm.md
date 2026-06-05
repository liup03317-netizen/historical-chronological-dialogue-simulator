# 高拟真打字机算法

## 核心问题

普通打字机动画速度恒定（如每 50ms 一个字），看起来机械呆板。本算法模拟人类打字的物理节奏：随机延时抖动 + 标点智能停顿。

## 算法实现

```typescript
import { useState, useEffect } from 'react';

interface TypewriterProps {
  figure: HistoricalFigureResponse;
  isActive: boolean;
  onComplete: () => void;
}

function Typewriter({ figure, isActive, onComplete }: TypewriterProps) {
  const [currentIndex, setCurrentIndex] = useState(0);

  useEffect(() => {
    if (!isActive) return;

    if (currentIndex < figure.text.length) {
      const char = figure.text[currentIndex];
      
      // 拟真阅读节奏间隔：85ms ~ 130ms（正常人跟读速度，约7-10字/秒）
      let delay = 85 + Math.random() * 45;
      
      // 根据特定字符类型执行智能拟真延时
      if (['.', '!', '?', '。', '！', '？'].includes(char)) {
        // 句尾呼吸停顿：500ms ~ 900ms
        delay = 500 + Math.random() * 400;
      } else if ([',', ';', '，', '；', ':', '：', '—', '-'].includes(char)) {
        // 句中喘息停顿：280ms ~ 450ms
        delay = 280 + Math.random() * 170;
      }

      const timeout = setTimeout(() => {
        setCurrentIndex(prev => prev + 1);
      }, delay);
      
      return () => clearTimeout(timeout);
    } else if (currentIndex >= figure.text.length && figure.isComplete) {
      // 打字完成，通知父组件
      onComplete();
    }
  }, [figure.text, currentIndex, isActive, figure.isComplete, onComplete]);

  // 当新文本追加时（流式更新），继续从当前位置打字
  // 无需重置 currentIndex，因为 useEffect 在 figure.text 变化时会重新评估

  return <span>{figure.text.slice(0, currentIndex)}</span>;
}
```

## 延时参数设计

| 字符类型 | 示例 | 延时范围 | 设计意图 |
|----------|------|----------|----------|
| 普通字符 | 汉字、字母 | 85-130ms | 正常人跟读速度，约7-10字/秒，含随机抖动 |
| 句末标点 | 。！？.!? | 500-900ms | 模拟人类视线停留与呼吸节奏 |
| 句中停顿 | ，；：—-,;:- | 280-450ms | 模拟语段间歇与思考微停 |

## React 状态管理集成

```typescript
// 父组件管理对话流程
function DialogueFlow({ figures }: { figures: HistoricalFigureResponse[] }) {
  const [activeIndex, setActiveIndex] = useState(0);

  const handleComplete = () => {
    // 当前人物打字完成，切换到下一位
    if (activeIndex < figures.length - 1) {
      setActiveIndex(prev => prev + 1);
    }
  };

  return (
    <div>
      {figures.map((figure, index) => (
        <div key={index} className="dialogue-bubble">
          {/* 头像、名称、时代标签 */}
          <Typewriter
            figure={figure}
            isActive={index === activeIndex}
            onComplete={handleComplete}
          />
        </div>
      ))}
    </div>
  );
}
```

## 流式更新适配

在渐进式解析场景中，`figure.text` 会随流式接收不断增长。关键处理：

1. **不重置 currentIndex**：新文本追加时，打字机从当前位置继续，不会从头开始
2. **isComplete 判断**：只有当 `figure.isComplete === true` 且打字完毕时才触发 `onComplete`
3. **清理 timeout**：组件卸载时必须清理 `setTimeout`，避免内存泄漏

## 性能优化

- 使用 `useCallback` 包裹 `onComplete` 回调，避免子组件不必要的重渲染
- 对于已完成的人物，直接渲染全文而非逐字打字，减少不必要的定时器开销
- 考虑使用 `requestAnimationFrame` 替代 `setTimeout` 在高频更新场景下的更优表现
