# Next.js 15 静态导出中实现动态路由多语言支持

Oct 12, 2025

> **关键词：** `Next.js 15` · `Static Export` · `Dynamic Routes` · `i18n` · `generateStaticParams` · `Server Components` · `Client Components` · `多语言`

---

**阅读时间：** 约 12 分钟

**难度等级：** ⭐⭐⭐ 中级

## 前言

在使用 Next.js 15 构建个人作品集网站时，实现多语言支持是常见需求。然而，当项目配置为静态导出模式（`output: "export"`）时，传统的服务端国际化方案（如 next-intl）无法使用，这给开发者带来了挑战。

本文将详细介绍如何在 Next.js 15 静态导出模式下，通过动态路由实现简洁优雅的多语言支持方案，包括完整的问题分析、解决方案和关键实现细节。

## 项目背景

**项目配置：**
- **框架：** Next.js 15 with App Router
- **输出模式：** Static Export（`output: "export"`）
- **部署平台：** GitHub Pages
- **UI 库：** HeroUI (基于 React Context)
- **数据源：** 本地 JSON 文件（`resume-zh.json`, `resume-en.json`）

**核心需求：**
- 支持中英文简历切换（`/resume/zh`, `/resume/en`）
- URL 路径清晰直观，便于分享
- 兼容静态导出，可部署到 GitHub Pages
- 提供友好的语言切换界面

## 问题分析

### 1. 静态导出的限制

Next.js 的静态导出模式有以下限制：

```typescript
// next.config.ts
const nextConfig = {
  output: "export",  // 启用静态导出
  images: {
    unoptimized: true  // 必须禁用图片优化
  }
}
```

**主要限制：**
- ❌ 不支持服务端 API Routes
- ❌ 不支持服务端渲染（SSR）
- ❌ 不支持基于 middleware 的国际化
- ❌ 动态路由需要 `generateStaticParams()`

### 2. Next.js 15 的新特性

在 Next.js 15 中，动态路由参数从同步变为异步：

```typescript
// Next.js 14 及更早版本
export default function Page({ params }: { params: { lang: string } }) {
  const lang = params.lang;  // 直接访问
}

// Next.js 15
export default async function Page({
  params
}: {
  params: Promise<{ lang: string }>
}) {
  const { lang } = await params;  // 需要 await
}
```

这个变化会导致直接迁移代码时出现运行时错误。

### 3. UI 库的客户端依赖

HeroUI 等现代 UI 库依赖 React Context，必须在客户端组件中使用：

```typescript
// ❌ 这样会报错
import { Link } from '@heroui/react';

export default async function Page() {
  return <Link href="/">Home</Link>;
  // Error: createContext only works in Client Components
}
```

## 解决方案设计

基于以上分析，采用**服务端组件 + 客户端组件混合架构**的方案：

### 架构设计

```
app/resume/
├── page.tsx              # 根路由重定向（Client Component）
├── [lang]/
│   ├── page.tsx         # 动态路由入口（Server Component）
│   └── ResumeContent.tsx # UI 渲染层（Client Component）
└── layout.tsx
```

**分层职责：**
1. **根路由** (`page.tsx`)：处理默认语言重定向
2. **服务端层** (`[lang]/page.tsx`)：处理数据加载和静态生成
3. **客户端层** (`ResumeContent.tsx`)：处理 UI 渲染和交互

## 完整实现步骤

### 步骤 1：创建动态路由结构

```bash
# 创建动态路由目录
mkdir -p app/resume/[lang]
```

**注意：** 目录名必须使用方括号 `[lang]`，这是 Next.js 的动态路由约定。

### 步骤 2：实现服务端组件

创建 `app/resume/[lang]/page.tsx`：

```typescript
import resumeZh from '@/config/data/resume-zh.json'
import resumeEn from '@/config/data/resume-en.json'
import ResumeContent from './ResumeContent'

// 关键：为静态导出生成参数列表
export function generateStaticParams() {
  return [
    { lang: 'zh' },
    { lang: 'en' },
  ]
}

// Next.js 15 中 params 是 Promise
export default async function ResumePage({
  params
}: {
  params: Promise<{ lang: string }>
}) {
  const { lang } = await params
  const resumeData = lang === 'en' ? resumeEn : resumeZh

  return <ResumeContent resumeData={resumeData} lang={lang} />
}
```

**关键点解析：**

1. **`generateStaticParams()`：** 告诉 Next.js 在构建时生成哪些静态页面
2. **`async` 函数：** Next.js 15 要求使用 `await params`
3. **数据预加载：** 在服务端完成数据选择，传递给客户端组件

### 步骤 3：实现客户端组件

创建 `app/resume/[lang]/ResumeContent.tsx`：

```typescript
'use client';
import { Link } from '@heroui/react'
import { getCommonIcon, getSocialIcon } from '@/lib/icons'
import { formatDateToShort, formatDateRange, formatDateToDetailed } from '@/lib/date-utils'

type ResumeData = {
  basics: any;
  skills: any[];
  work: any[];
  projects: any[];
  education: any[];
  languages: any[];
}

export default function ResumeContent({
  resumeData,
  lang
}: {
  resumeData: ResumeData,
  lang: string
}) {
  const { basics, skills, work, projects, education, languages } = resumeData
  const isEnglish = lang === 'en'

  return (
    <div className="max-w-6xl mx-auto px-8 py-10 print:py-0">
      {/* 语言切换器 */}
      <div className="flex justify-end gap-3 mb-6 no-print">
        <Link
          href="/resume/zh"
          className={`text-sm ${
            !isEnglish
              ? 'font-bold text-text-primary'
              : 'text-text-primary opacity-60 hover:opacity-100'
          } transition-opacity`}
        >
          中文
        </Link>
        <span className="text-text-primary opacity-40">|</span>
        <Link
          href="/resume/en"
          className={`text-sm ${
            isEnglish
              ? 'font-bold text-text-primary'
              : 'text-text-primary opacity-60 hover:opacity-100'
          } transition-opacity`}
        >
          English
        </Link>
      </div>

      {/* 简历内容 */}
      <div className="grid grid-cols-1 md:grid-cols-10">
        {/* 左侧栏 */}
        <div className="md:col-span-2">
          <h1 className="text-3xl text-black mb-1">
            {basics.name}
          </h1>
          <h2 className="text-md font-light text-text-primary opacity-60 mb-1">
            {basics.label}
          </h2>
          {/* 更多内容... */}
        </div>

        {/* 右侧内容 */}
        <div className="md:col-span-6">
          {/* 各个板块... */}
        </div>
      </div>
    </div>
  )
}
```

**关键点解析：**

1. **`'use client'` 指令：** 声明为客户端组件，可使用 React Context
2. **Props 传递：** 从服务端接收数据和语言参数
3. **条件样式：** 根据当前语言高亮对应的切换按钮

### 步骤 4：实现根路由重定向

创建 `app/resume/page.tsx`：

```typescript
'use client';
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function Resume() {
  const router = useRouter();

  useEffect(() => {
    // 默认重定向到中文版
    router.replace('/resume/zh');
  }, [router]);

  return (
    <div className="flex items-center justify-center min-h-screen">
      <p className="text-text-primary opacity-60">Redirecting...</p>
    </div>
  );
}
```

**说明：** 当用户访问 `/resume` 时，自动跳转到 `/resume/zh`。

## 常见问题与解决方案

### 问题 1：缺少 generateStaticParams 错误

**错误信息：**
```
Error: Page "/resume/[lang]/page" is missing exported function
"generateStaticParams()", which is required with "output: export" config.
```

**解决方案：**
必须导出 `generateStaticParams` 函数：

```typescript
export function generateStaticParams() {
  return [
    { lang: 'zh' },
    { lang: 'en' },
  ]
}
```

### 问题 2：createContext 错误

**错误信息：**
```
TypeError: createContext only works in Client Components.
Add the "use client" directive at the top of the file to use it.
```

**原因分析：**
在服务端组件中直接使用了需要 React Context 的 UI 库组件（如 HeroUI 的 `Link`）。

**解决方案：**
分离服务端和客户端组件：
- 服务端组件：负责数据加载
- 客户端组件：负责 UI 渲染

### 问题 3：params 类型错误

**错误信息：**
```
Property 'lang' does not exist on type 'Promise<{ lang: string }>'
```

**解决方案：**
在 Next.js 15 中，需要 await params：

```typescript
// ❌ 错误写法
export default function Page({ params }: { params: { lang: string } }) {
  const lang = params.lang;
}

// ✅ 正确写法
export default async function Page({
  params
}: {
  params: Promise<{ lang: string }>
}) {
  const { lang } = await params;
}
```

### 问题 4：构建时缓存问题

如果修改代码后仍然报错，可能是缓存问题：

```bash
# 清除 Next.js 缓存并重启
rm -rf .next
npm run dev
```

## 性能优化建议

### 1. 静态生成优化

通过 `generateStaticParams`，构建时会生成所有语言版本的静态 HTML：

```bash
# 构建输出示例
out/
├── resume/
│   ├── zh.html          # 预生成的中文版
│   ├── en.html          # 预生成的英文版
│   └── index.html       # 重定向页面
```

### 2. 代码分割

客户端组件会自动进行代码分割：

```typescript
// ResumeContent.tsx 会被打包为独立的 chunk
// 只有访问简历页面时才会加载
```

### 3. 字体优化

使用 CSS 变量统一管理文字颜色，方便调整：

```css
/* styles/globals.css */
:root {
  --text-primary: #374151;  /* 柔和的深灰色 */
}

.text-text-primary {
  color: var(--text-primary);
}
```

## 方案对比分析

### 本方案 vs URL 参数方案

| 特性 | 动态路由方案 | URL 参数方案 |
|------|------------|------------|
| URL 格式 | `/resume/zh` | `/resume?lang=zh` |
| SEO 友好度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 静态预生成 | ✅ 支持 | ❌ 单页面 |
| 实现复杂度 | 中等 | 简单 |
| 分享友好度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 本方案 vs 子目录方案

| 特性 | 动态路由方案 | 子目录方案 |
|------|------------|-----------|
| 代码复用 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 维护成本 | 低 | 中等 |
| 文件结构 | `/resume/[lang]` | `/zh/resume`, `/en/resume` |
| 扩展性 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

## 扩展思路

### 1. 添加更多语言

只需更新 `generateStaticParams`：

```typescript
export function generateStaticParams() {
  return [
    { lang: 'zh' },
    { lang: 'en' },
    { lang: 'ja' },  // 日语
    { lang: 'ko' },  // 韩语
  ]
}
```

### 2. 语言检测与自动跳转

在根路由中添加浏览器语言检测：

```typescript
'use client';
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function Resume() {
  const router = useRouter();

  useEffect(() => {
    // 检测浏览器语言
    const browserLang = navigator.language.startsWith('en') ? 'en' : 'zh';
    router.replace(`/resume/${browserLang}`);
  }, [router]);

  return <div>Redirecting...</div>;
}
```

### 3. 集成到更大的项目

将该模式应用到整个网站：

```
app/
├── [lang]/
│   ├── page.tsx          # 首页
│   ├── about/
│   │   └── page.tsx      # 关于页
│   ├── projects/
│   │   └── page.tsx      # 项目页
│   └── resume/
│       └── page.tsx      # 简历页
```

## 总结

本文介绍了在 Next.js 15 静态导出模式下实现多语言支持的完整方案。通过合理的架构设计，成功解决了静态导出的限制、Next.js 15 的 API 变化、以及 UI 库的客户端依赖等多个技术挑战。

**核心要点：**
1. 使用动态路由 `[lang]` 实现清晰的 URL 结构
2. 通过 `generateStaticParams()` 支持静态导出
3. 采用服务端组件 + 客户端组件分离架构
4. 正确处理 Next.js 15 的异步 params

**适用场景：**
- 需要静态部署的多语言网站（GitHub Pages、Netlify 等）
- SEO 要求较高的项目
- 内容相对稳定的展示型网站

**源码参考：**
完整的实现代码可参考项目仓库中的 `app/resume/[lang]` 目录。

---

**相关文章：**
- [解决 Next.js 静态导出 PDF 中文乱码问题](./004_解决%20Next.js%20静态导出%20PDF%20中文乱码问题.md)

**参考资料：**
- [Next.js Dynamic Routes 文档](https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes)
- [Next.js Static Exports 文档](https://nextjs.org/docs/app/building-your-application/deploying/static-exports)
- [Next.js 15 发布说明](https://nextjs.org/blog/next-15)
