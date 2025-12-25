# 解决 Next.js 静态导出 PDF 中文乱码问题

Oct 11, 2025

> **关键词：** `Next.js` · `Font Optimization` · `PDF Generation` · `Puppeteer` · `Google Fonts` · `中文字体` · `CSS Variables`

---

## 前言

在使用 Next.js 15 构建个人简历网站时，通过 Puppeteer 将 HTML 页面导出为 PDF 格式是常见需求。然而，当页面中包含中文内容时，生成的 PDF 常常出现乱码或方块字符，而在浏览器中预览却显示正常。

本文将深入分析问题根源，并提供完整的解决方案，帮助开发者在 Next.js 项目中实现中英文混排的 PDF 导出。

## 问题场景

项目配置：
- **框架：** Next.js 15 with App Router
- **输出模式：** Static Export（`output: "export"`）
- **PDF 生成：** Puppeteer + Chromium
- **字体加载：** Next.js Font Optimization

初始状态下，根布局使用了 `Nunito_Sans` 字体：

```tsx
// app/layout.tsx
import { Nunito_Sans } from 'next/font/google';

const nunito_sans = Nunito_Sans({
  subsets: ['latin'],
  variable: '--font-nunito-sans',
  display: 'swap'
});

export default function RootLayout({ children }) {
  return (
    <html lang="zh">
      <body className={nunito_sans.className}>
        {children}
      </body>
    </html>
  );
}
```

**问题表现：**
- 浏览器预览：中英文正常显示
- PDF 导出：英文正常，中文显示为 ▯▯▯ 或乱码

## 问题分析

### 根本原因

`Nunito_Sans` 是一款拉丁字体，字符集中不包含中文字形（Glyphs）。当 Puppeteer 将页面渲染为 PDF 时，无法找到对应的中文字符映射，导致显示失败。

浏览器能正常显示的原因是：浏览器会自动回退到系统默认字体（如微软雅黑、苹方等），而 Puppeteer 在无头模式下没有这些系统字体，必须显式指定。

### 为什么不能直接用两个 className？

尝试同时应用两个字体：

```tsx
// ❌ 错误示范
<body className={`${nunito_sans.className} ${notoSansSC.className}`}>
```

这种方式无效，因为后面的 CSS 类会完全覆盖前面的 `font-family` 属性，最终只有 `notoSansSC` 生效。

## 解决方案

使用 Next.js 字体系统的 **`variable` 模式**，结合 Tailwind CSS 的字体栈配置，实现字符级别的字体回退。

### 步骤 1：配置字体变量

在 `app/layout.tsx` 中同时加载英文和中文字体：

```tsx
import { Nunito_Sans, Noto_Sans_SC } from 'next/font/google';

// 英文字体
const nunito_sans = Nunito_Sans({
  subsets: ['latin'],
  variable: '--font-nunito-sans',
  display: 'swap'
});

// 中文字体
const notoSansSC = Noto_Sans_SC({
  subsets: ['latin'],
  weight: ['300', '400', '500', '700'],
  variable: '--font-noto-sans-sc',
  display: 'swap'
});

export default function RootLayout({ children }) {
  return (
    <html lang="zh" suppressHydrationWarning>
      <body className={`${nunito_sans.variable} ${notoSansSC.variable} font-sans text-md`}>
        <HeroUIProvider>
          <ThemeProvider>
            {children}
          </ThemeProvider>
        </HeroUIProvider>
      </body>
    </html>
  );
}
```

### 步骤 2：配置 Tailwind 字体栈

在 `tailwind.config.js` 中定义字体回退顺序：

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  theme: {
    extend: {
      fontFamily: {
        sans: [
          'var(--font-nunito-sans)',  // 优先使用英文字体
          'var(--font-noto-sans-sc)',  // 回退到中文字体
          'system-ui',
          'sans-serif'
        ],
      },
    },
  },
  plugins: [],
};
```

### 步骤 3：验证效果

重新构建并生成 PDF：

```bash
npm run build
npm run export  # 运行 Puppeteer 脚本生成 PDF
```

打开生成的 `resume.pdf`，中英文应均正常显示。

## 技术原理深度解析

### className vs Variable 模式

Next.js Font Optimization 提供两种字体应用方式：

#### 1. className 模式（直接应用）

```tsx
const font = Font_Name({ subsets: ['latin'] });
<body className={font.className}>
```

生成的 CSS：

```css
.__className_a1b2c3 {
  font-family: 'Font Name', sans-serif;
}
```

特点：直接设置 `font-family`，多个 className 会冲突。

#### 2. Variable 模式（CSS 变量）

```tsx
const font = Font_Name({
  subsets: ['latin'],
  variable: '--font-name'  // 定义 CSS 变量
});
<body className={font.variable}>
```

生成的 CSS：

```css
.__variable_a1b2c3 {
  --font-name: 'Font Name', sans-serif;
}
```

特点：只定义 CSS 变量，不直接设置字体，多个 variable 可以共存。

### 字体回退机制

当在 Tailwind 配置中使用 `font-sans` 时：

```css
/* 展开后的 CSS */
body {
  font-family:
    var(--font-nunito-sans),   /* 步骤 1 */
    var(--font-noto-sans-sc),  /* 步骤 2 */
    system-ui,                 /* 步骤 3 */
    sans-serif;                /* 步骤 4 */
}
```

浏览器渲染流程：

1. **尝试 Nunito Sans**
   - 英文字符 A-Z：✅ 存在 → 使用此字体渲染
   - 中文字符 "你好"：❌ 不存在 → 跳过，继续下一个

2. **回退到 Noto Sans SC**
   - 中文字符 "你好"：✅ 存在 → 使用此字体渲染
   - 英文字符 A-Z：✅ 也存在，但已被前面处理

3. **最终效果**
   - 英文：Nunito Sans（优雅的拉丁字体）
   - 中文：Noto Sans SC（思源黑体）

这种机制称为 **字符级回退（Per-Character Fallback）**，是 CSS 的标准行为。

## 完整工作流程图示

```
┌─────────────────────────────────────────────────────────┐
│ Next.js 字体系统                                          │
│ ┌─────────────────┐      ┌─────────────────┐            │
│ │ Nunito_Sans     │      │ Noto_Sans_SC    │            │
│ │ variable mode   │      │ variable mode   │            │
│ └────────┬────────┘      └────────┬────────┘            │
│          │                        │                      │
│          └─────────┬──────────────┘                      │
└────────────────────┼─────────────────────────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ CSS Custom Properties│
          │ --font-nunito-sans   │
          │ --font-noto-sans-sc  │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ Tailwind Config      │
          │ fontFamily.sans: [   │
          │   'var(--font-...)', │
          │ ]                    │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ Browser Rendering    │
          │ - 英文用 Nunito Sans │
          │ - 中文用 Noto Sans SC│
          └──────────────────────┘
```

## 常见问题

### Q1: 为什么不在 @media print 中设置字体？

在 `globals.css` 中添加打印样式是可行的：

```css
@media print {
  @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@300;400;500;700&display=swap');

  body {
    font-family: 'Noto Sans SC', sans-serif !important;
  }
}
```

但这种方式有两个缺点：
1. **外部依赖：** 需要 Puppeteer 能访问 Google Fonts CDN
2. **覆盖问题：** 英文也会使用中文字体，失去了字体层次感

使用 Next.js Font Optimization 的优势：
- 字体在构建时自动下载并自托管
- 无外部网络依赖
- 支持字符级回退，保持字体多样性

### Q2: 可以只用一个中文字体吗？

可以，但不推荐：

```tsx
const notoSansSC = Noto_Sans_SC({
  subsets: ['latin'],
  weight: ['300', '400', '500', '700'],
});

<body className={notoSansSC.className}>
```

缺点：
- Noto Sans SC 的拉丁字符设计相对平庸
- 失去了英文字体的视觉层次
- 文件体积更大（包含了拉丁字符）

最佳实践是使用专业的英文字体 + 专业的中文字体。

### Q3: 支持哪些中文字体？

Google Fonts 提供的常用中文字体：

| 字体名称 | 风格 | 适用场景 |
|---------|------|---------|
| **Noto Sans SC** | 无衬线 | 现代、简洁、适合屏幕阅读 |
| **Noto Serif SC** | 衬线 | 传统、优雅、适合正式文档 |
| **ZCOOL XiaoWei** | 艺术体 | 个性化设计 |
| **Ma Shan Zheng** | 手写体 | 轻松、亲切感 |

选择建议：
- **简历/文档：** Noto Sans SC（本文方案）
- **博客/文章：** Noto Serif SC
- **创意项目：** ZCOOL 系列

## 项目实战建议

### 1. 字重（Font Weight）选择

Noto Sans SC 提供 9 个字重（100-900），建议按需加载：

```tsx
const notoSansSC = Noto_Sans_SC({
  subsets: ['latin'],
  weight: ['400', '700'],  // 只加载常规和粗体
  display: 'swap'
});
```

字重对应关系：
- 300: Light
- 400: Regular（正文）
- 500: Medium
- 700: Bold（标题）

### 2. 性能优化

使用 `display: 'swap'` 策略：

```tsx
const font = Font_Name({
  display: 'swap',  // 字体加载期间先显示备用字体
});
```

其他选项：
- `block`: 短暂隐藏文本（最多 3s）
- `fallback`: 极短隐藏（100ms）后使用备用字体
- `optional`: 仅在缓存时使用自定义字体

推荐 `swap`，避免 FOIT（Flash of Invisible Text）。

### 3. 构建体积优化

字体文件通常较大，Next.js 会自动进行优化：

- 自动子集化（Subsetting）
- 按需加载（Only used characters）
- 预连接 Google Fonts（Preconnect）

查看构建输出：

```bash
npm run build

# 输出示例
Route (app)                Size     First Load JS
┌ ○ /                      5.2 kB          92 kB
└ ○ /resume               8.1 kB          95 kB
+ First Load JS shared    87 kB
  ├ chunks/...
  └ fonts/NunitoSans.woff2  45 kB
  └ fonts/NotoSansSC.woff2  120 kB  # 中文字体较大
```

## 延伸思考

### 多语言支持

如果项目需要支持多种语言：

```tsx
const inter = Inter({ subsets: ['latin'], variable: '--font-inter' });
const notoSansSC = Noto_Sans_SC({ subsets: ['latin'], variable: '--font-sc' });
const notoSansJP = Noto_Sans_JP({ subsets: ['latin'], variable: '--font-jp' });

// Tailwind Config
fontFamily: {
  sans: [
    'var(--font-inter)',
    'var(--font-sc)',
    'var(--font-jp)',
    'sans-serif'
  ]
}
```

浏览器会根据字符自动选择合适的字体。

### 与其他 PDF 生成方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| **Puppeteer + HTML** | 样式还原度高，所见即所得 | 需要 Chromium，体积大 |
| **jsPDF** | 纯 JS，体积小 | 中文支持需手动处理字体 |
| **React-PDF** | React 生态，组件化 | 学习成本高，样式受限 |
| **Prince XML** | 专业排版，商业级 | 付费，价格昂贵 |

本文方案基于 Puppeteer，适合已有 HTML/CSS 的场景。

## 总结

解决 Next.js PDF 中文乱码的核心要点：

1. **使用 `variable` 模式加载字体**，避免 className 冲突
2. **在 Tailwind 中配置字体栈**，实现字符级回退
3. **英文字体在前，中文字体在后**，保持视觉层次
4. **选择合适的中文字体**（推荐 Noto Sans SC）
5. **按需加载字重**，控制文件体积

完整示例代码：[sggmico.github.io](https://github.com/sggmico/sggmico.github.io)

## 参考资料

- [Next.js Font Optimization](https://nextjs.org/docs/app/building-your-application/optimizing/fonts)
- [CSS Font Stack MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/font-family)
- [Google Fonts](https://fonts.google.com/)
- [Noto Sans CJK](https://github.com/googlefonts/noto-cjk)
- [Puppeteer Documentation](https://pptr.dev/)

---

**最后更新：** 2025-10-11
