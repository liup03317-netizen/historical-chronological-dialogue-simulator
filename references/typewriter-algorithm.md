# 高拟真打字机算法

## 核心问题

普通打字机动画速度恒定（如每 50ms 一个字），看起来机械呆板。本算法模拟人类打字的物理节奏：随机延时抖动 + 标点智能停顿。

## 算法实现（纯原生 JS，零框架依赖）

```javascript
function startTypewriter(text, figureIdx) {
  var currentCharIndex = 0;

  function typeNextChar() {
    if (currentCharIndex >= text.length) {
      // 打字完成，隐藏光标
      var cursor = document.getElementById('typewriter-cursor-' + figureIdx);
      if (cursor) cursor.style.display = 'none';

      // 链式驱动：延迟后添加下一位人物
      setTimeout(function() {
        currentFigureIndex++;
        if (currentFigureIndex < FIGURES_DATA.length) {
          renderFigureBubble(FIGURES_DATA[currentFigureIndex]);
        } else {
          renderSummary();
        }
      }, 600);
      return;
    }

    var char = text[currentCharIndex];
    var textEl = document.getElementById('typewriter-text-' + figureIdx);
    if (textEl) textEl.textContent += char;

    currentCharIndex++;
    scrollToBottom();

    // 拟真阅读节奏间隔：85ms ~ 130ms（正常人跟读速度，约7-10字/秒）
    var delay = 85 + Math.random() * 45;

    // 根据特定字符类型执行智能拟真延时
    if (['.', '!', '?', '。', '！', '？'].indexOf(char) !== -1) {
      // 句尾呼吸停顿：500ms ~ 900ms
      delay = 500 + Math.random() * 400;
    } else if ([',', ';', '，', '；', ':', '：', '—', '-'].indexOf(char) !== -1) {
      // 句中喘息停顿：280ms ~ 450ms
      delay = 280 + Math.random() * 170;
    }

    setTimeout(typeNextChar, delay);
  }

  typeNextChar();
}
```

## 延时参数设计

| 字符类型 | 示例 | 延时范围 | 设计意图 |
|----------|------|----------|----------|
| 普通字符 | 汉字、字母 | 85-130ms | 正常人跟读速度，约7-10字/秒，含随机抖动 |
| 句末标点 | 。！？.!? | 500-900ms | 模拟人类视线停留与呼吸节奏 |
| 句中停顿 | ，；：—-,;:- | 280-450ms | 模拟语段间歇与思考微停 |

## 链式驱动机制

打字机完成当前人物后，通过 `setTimeout` 延迟 600ms 再添加下一位人物，形成仪式感的渐进式体验：

```
第一位人物入场 → 打字机逐字渲染 → 完成 → (600ms 仪式感延迟) → 第二位人物入场 → ... → 总结卡片出现
```

## 框架无关设计

本算法使用纯原生 JS 实现，**不依赖 React、Vue 或任何前端框架**：

- 使用 `document.getElementById` 操作 DOM
- 使用 `setTimeout` 实现延时调度
- 使用 `textContent +=` 逐字追加文本
- 使用 `scrollToBottom()` 自动滚动

### 对比 React 方案

| 维度 | React 方案（旧） | 原生 JS 方案（新） |
|------|-------------------|---------------------|
| 框架依赖 | React + ReactDOM (~260KB) | 无 |
| 编译依赖 | Babel Standalone (~900KB) | 无 |
| CSS 依赖 | Tailwind CDN (~400KB) | 无 |
| CDN 请求数 | 5 个 | 0 个 |
| 离线可用 | ❌ | ✅ |
| 加载时间 | 2-5 秒 | <100ms |

## 性能优化

- 对于已完成的人物，直接渲染全文而非逐字打字
- 每位人物独立管理自己的定时器，互不干扰
- 页面卸载时无需手动清理（非 React 组件生命周期）
- 使用 `textContent` 而非 `innerHTML` 避免重复解析 HTML
