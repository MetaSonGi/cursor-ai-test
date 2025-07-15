# 🚀 개발 로드맵

## Phase 1: 프로젝트 초기 설정 및 기본 구조 (1-2주)

### 1.1 개발 환경 설정

#### 1.1.1 Next.js 프로젝트 생성
```bash
# Next.js v15 프로젝트 생성
npx create-next-app@latest notion-blog --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"

# 프로젝트 디렉토리 이동
cd notion-blog

# 의존성 설치 확인
npm install
```

#### 1.1.2 기본 의존성 설치
```bash
# UI 컴포넌트 라이브러리
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-slot
npm install class-variance-authority clsx tailwind-merge

# Notion API
npm install @notionhq/client

# 마크다운 처리
npm install react-markdown remark-gfm rehype-highlight

# 유틸리티
npm install date-fns

# 개발 도구
npm install -D @types/node @types/react @types/react-dom
```

#### 1.1.3 ShadcnUI 초기 설정
```bash
# ShadcnUI 초기화
npx shadcn@latest init

# 기본 컴포넌트 설치
npx shadcn@latest add button card input dialog dropdown-menu
```

### 1.2 프로젝트 구조 설정

#### 1.2.1 디렉토리 구조 생성
```
src/
├── app/
│   ├── (routes)/
│   │   ├── blog/
│   │   │   ├── [slug]/
│   │   │   │   └── page.tsx
│   │   │   └── page.tsx
│   │   ├── category/
│   │   │   └── [category]/
│   │   │       └── page.tsx
│   │   └── tag/
│   │       └── [tag]/
│   │           └── page.tsx
│   ├── api/
│   │   └── notion/
│   │       ├── posts/
│   │       │   └── route.ts
│   │       ├── categories/
│   │       │   └── route.ts
│   │       └── tags/
│   │           └── route.ts
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── ui/
│   ├── blog/
│   │   ├── PostCard.tsx
│   │   ├── PostList.tsx
│   │   ├── PostContent.tsx
│   │   ├── TableOfContents.tsx
│   │   └── SearchBar.tsx
│   └── layout/
│       ├── Header.tsx
│       ├── Footer.tsx
│       └── Navigation.tsx
├── lib/
│   ├── notion.ts
│   ├── utils.ts
│   └── constants.ts
├── types/
│   └── notion.ts
└── hooks/
    ├── useNotionData.ts
    └── useSearch.ts
```

#### 1.2.2 환경 변수 설정
```bash
# .env.local 파일 생성
touch .env.local
```

```env
# .env.local
NOTION_TOKEN=your_notion_integration_token
NOTION_DATABASE_ID=your_database_id
GITHUB_REPO=username/repo-name
GITHUB_REPO_ID=repo-id
GITHUB_CATEGORY_ID=category-id
```

### 1.3 기본 타입 정의

#### 1.3.1 Notion 타입 정의
```typescript
// src/types/notion.ts
export interface BlogPost {
  id: string;
  title: string;
  slug: string;
  content: string;
  excerpt: string;
  publishedAt: string;
  updatedAt: string;
  tags: string[];
  category: string;
  coverImage?: string;
  isPublished: boolean;
}

export interface NotionPage {
  id: string;
  properties: {
    title: { title: Array<{ plain_text: string }> };
    slug: { rich_text: Array<{ plain_text: string }> };
    excerpt: { rich_text: Array<{ plain_text: string }> };
    tags: { multi_select: Array<{ name: string }> };
    category: { select: { name: string } };
    publishedAt: { date: { start: string } };
    isPublished: { checkbox: boolean };
  };
}
```

### 1.4 기본 레이아웃 구현

#### 1.4.1 루트 레이아웃
```typescript
// src/app/layout.tsx
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'
import Header from '@/components/layout/Header'
import Footer from '@/components/layout/Footer'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'Notion Blog',
  description: '개인 블로그',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ko">
      <body className={inter.className}>
        <div className="min-h-screen flex flex-col">
          <Header />
          <main className="flex-1">
            {children}
          </main>
          <Footer />
        </div>
      </body>
    </html>
  )
}
```

## Phase 2: Notion API 연동 및 데이터 페칭 (1-2주)

### 2.1 Notion API 클라이언트 설정

#### 2.1.1 Notion 클라이언트 구현
```typescript
// src/lib/notion.ts
import { Client } from '@notionhq/client'

const notion = new Client({
  auth: process.env.NOTION_TOKEN,
})

export const getDatabase = async () => {
  const response = await notion.databases.query({
    database_id: process.env.NOTION_DATABASE_ID!,
    filter: {
      property: 'isPublished',
      checkbox: {
        equals: true,
      },
    },
    sorts: [
      {
        property: 'publishedAt',
        direction: 'descending',
      },
    ],
  })
  return response.results
}

export const getPage = async (pageId: string) => {
  const response = await notion.pages.retrieve({
    page_id: pageId,
  })
  return response
}

export const getPageBlocks = async (pageId: string) => {
  const response = await notion.blocks.children.list({
    block_id: pageId,
  })
  return response.results
}
```

### 2.2 API 라우트 구현

#### 2.2.1 포스트 목록 API
```typescript
// src/app/api/notion/posts/route.ts
import { NextResponse } from 'next/server'
import { getDatabase } from '@/lib/notion'

export async function GET() {
  try {
    const pages = await getDatabase()
    const posts = pages.map((page: any) => ({
      id: page.id,
      title: page.properties.title.title[0]?.plain_text || '',
      slug: page.properties.slug.rich_text[0]?.plain_text || '',
      excerpt: page.properties.excerpt.rich_text[0]?.plain_text || '',
      publishedAt: page.properties.publishedAt.date?.start || '',
      tags: page.properties.tags.multi_select.map((tag: any) => tag.name),
      category: page.properties.category.select?.name || '',
    }))

    return NextResponse.json(posts)
  } catch (error) {
    return NextResponse.json({ error: 'Failed to fetch posts' }, { status: 500 })
  }
}
```

### 2.3 데이터 페칭 훅 구현

#### 2.3.1 커스텀 훅
```typescript
// src/hooks/useNotionData.ts
import { useState, useEffect } from 'react'
import { BlogPost } from '@/types/notion'

export const useNotionData = () => {
  const [posts, setPosts] = useState<BlogPost[]>([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const fetchPosts = async () => {
      try {
        const response = await fetch('/api/notion/posts')
        const data = await response.json()
        setPosts(data)
      } catch (err) {
        setError('포스트를 불러오는데 실패했습니다.')
      } finally {
        setLoading(false)
      }
    }

    fetchPosts()
  }, [])

  return { posts, loading, error }
}
```

## Phase 3: 기본 UI 컴포넌트 구현 (1주)

### 3.1 레이아웃 컴포넌트

#### 3.1.1 헤더 컴포넌트
```typescript
// src/components/layout/Header.tsx
'use client'

import Link from 'next/link'
import { Button } from '@/components/ui/button'
import { SearchBar } from '@/components/blog/SearchBar'

export default function Header() {
  return (
    <header className="border-b">
      <div className="container mx-auto px-4 py-4 flex items-center justify-between">
        <Link href="/" className="text-2xl font-bold">
          Notion Blog
        </Link>
        
        <nav className="hidden md:flex items-center space-x-6">
          <Link href="/blog" className="hover:text-gray-600">
            블로그
          </Link>
          <Link href="/category" className="hover:text-gray-600">
            카테고리
          </Link>
          <SearchBar />
        </nav>

        <Button variant="outline" size="sm">
          다크모드
        </Button>
      </div>
    </header>
  )
}
```

### 3.2 블로그 컴포넌트

#### 3.2.1 포스트 카드 컴포넌트
```typescript
// src/components/blog/PostCard.tsx
import Link from 'next/link'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { BlogPost } from '@/types/notion'
import { format } from 'date-fns'

interface PostCardProps {
  post: BlogPost
}

export default function PostCard({ post }: PostCardProps) {
  return (
    <Card className="hover:shadow-lg transition-shadow">
      <CardHeader>
        <CardTitle className="text-xl">
          <Link href={`/blog/${post.slug}`} className="hover:text-blue-600">
            {post.title}
          </Link>
        </CardTitle>
        <div className="text-sm text-gray-500">
          {format(new Date(post.publishedAt), 'yyyy년 MM월 dd일')}
        </div>
      </CardHeader>
      <CardContent>
        <p className="text-gray-600 mb-4">{post.excerpt}</p>
        <div className="flex flex-wrap gap-2">
          {post.tags.map((tag) => (
            <span
              key={tag}
              className="px-2 py-1 bg-gray-100 text-sm rounded"
            >
              {tag}
            </span>
          ))}
        </div>
      </CardContent>
    </Card>
  )
}
```

## Phase 4: 첫 번째 배포 (1주)

### 4.1 Vercel 배포 준비

#### 4.1.1 Vercel 설정
```bash
# Vercel CLI 설치
npm install -g vercel

# Vercel 로그인
vercel login

# 프로젝트 배포
vercel
```

#### 4.1.2 환경 변수 설정
- Vercel 대시보드에서 환경 변수 설정
- Notion API 토큰, 데이터베이스 ID 등 설정

### 4.2 기본 페이지 구현

#### 4.2.1 홈페이지
```typescript
// src/app/page.tsx
import { PostList } from '@/components/blog/PostList'

export default function HomePage() {
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-4xl font-bold mb-8">최신 포스트</h1>
      <PostList />
    </div>
  )
}
```

#### 4.2.2 블로그 목록 페이지
```typescript
// src/app/(routes)/blog/page.tsx
import { PostList } from '@/components/blog/PostList'

export default function BlogPage() {
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">블로그</h1>
      <PostList />
    </div>
  )
}
```

## Phase 5: 고급 기능 구현 (2-3주)

### 5.1 검색 기능 구현

#### 5.1.1 검색 컴포넌트
```typescript
// src/components/blog/SearchBar.tsx
'use client'

import { useState } from 'react'
import { Input } from '@/components/ui/input'
import { Button } from '@/components/ui/button'
import { Search } from 'lucide-react'

export function SearchBar() {
  const [query, setQuery] = useState('')

  const handleSearch = (e: React.FormEvent) => {
    e.preventDefault()
    // 검색 로직 구현
  }

  return (
    <form onSubmit={handleSearch} className="flex items-center space-x-2">
      <Input
        type="text"
        placeholder="검색..."
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        className="w-64"
      />
      <Button type="submit" size="sm">
        <Search className="h-4 w-4" />
      </Button>
    </form>
  )
}
```

### 5.2 마크다운 렌더링

#### 5.2.1 포스트 콘텐츠 컴포넌트
```typescript
// src/components/blog/PostContent.tsx
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import rehypeHighlight from 'rehype-highlight'

interface PostContentProps {
  content: string
}

export default function PostContent({ content }: PostContentProps) {
  return (
    <div className="prose prose-lg max-w-none">
      <ReactMarkdown
        remarkPlugins={[remarkGfm]}
        rehypePlugins={[rehypeHighlight]}
      >
        {content}
      </ReactMarkdown>
    </div>
  )
}
```

### 5.3 목차 자동 생성

#### 5.3.1 목차 컴포넌트
```typescript
// src/components/blog/TableOfContents.tsx
'use client'

import { useEffect, useState } from 'react'

interface TocItem {
  id: string
  text: string
  level: number
}

export default function TableOfContents() {
  const [toc, setToc] = useState<TocItem[]>([])

  useEffect(() => {
    const headings = document.querySelectorAll('h1, h2, h3, h4, h5, h6')
    const tocItems: TocItem[] = []

    headings.forEach((heading) => {
      const level = parseInt(heading.tagName.charAt(1))
      const text = heading.textContent || ''
      const id = heading.id

      tocItems.push({ id, text, level })
    })

    setToc(tocItems)
  }, [])

  return (
    <nav className="space-y-2">
      <h3 className="font-semibold mb-4">목차</h3>
      {toc.map((item) => (
        <a
          key={item.id}
          href={`#${item.id}`}
          className={`block text-sm hover:text-blue-600 ${
            item.level === 1 ? 'font-medium' : 'ml-4'
          }`}
        >
          {item.text}
        </a>
      ))}
    </nav>
  )
}
```

## Phase 6: SEO 최적화 및 성능 개선 (1주)

### 6.1 메타데이터 최적화

#### 6.1.1 동적 메타데이터
```typescript
// src/app/(routes)/blog/[slug]/page.tsx
import { Metadata } from 'next'
import { getPost } from '@/lib/notion'

interface PostPageProps {
  params: { slug: string }
}

export async function generateMetadata({ params }: PostPageProps): Promise<Metadata> {
  const post = await getPost(params.slug)
  
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: post.coverImage ? [post.coverImage] : [],
    },
  }
}
```

### 6.2 이미지 최적화

#### 6.2.1 Next.js Image 컴포넌트 활용
```typescript
// src/components/blog/PostImage.tsx
import Image from 'next/image'

interface PostImageProps {
  src: string
  alt: string
  width: number
  height: number
}

export default function PostImage({ src, alt, width, height }: PostImageProps) {
  return (
    <div className="my-8">
      <Image
        src={src}
        alt={alt}
        width={width}
        height={height}
        className="rounded-lg"
        priority={false}
      />
    </div>
  )
}
```

## Phase 7: 댓글 시스템 구현 (1주)

### 7.1 Giscus 설정

#### 7.1.1 댓글 컴포넌트
```typescript
// src/components/blog/Comments.tsx
'use client'

import { useEffect } from 'react'

interface CommentsProps {
  slug: string
}

export default function Comments({ slug }: CommentsProps) {
  useEffect(() => {
    const script = document.createElement('script')
    script.src = 'https://giscus.app/client.js'
    script.setAttribute('data-repo', process.env.NEXT_PUBLIC_GITHUB_REPO!)
    script.setAttribute('data-repo-id', process.env.NEXT_PUBLIC_GITHUB_REPO_ID!)
    script.setAttribute('data-category', 'Announcements')
    script.setAttribute('data-category-id', process.env.NEXT_PUBLIC_GITHUB_CATEGORY_ID!)
    script.setAttribute('data-mapping', 'pathname')
    script.setAttribute('data-strict', '0')
    script.setAttribute('data-reactions-enabled', '1')
    script.setAttribute('data-emit-metadata', '0')
    script.setAttribute('data-input-position', 'bottom')
    script.setAttribute('data-theme', 'preferred_color_scheme')
    script.setAttribute('data-lang', 'ko')
    script.crossOrigin = 'anonymous'
    script.async = true

    const giscusContainer = document.getElementById('giscus-container')
    if (giscusContainer) {
      giscusContainer.appendChild(script)
    }

    return () => {
      if (giscusContainer) {
        giscusContainer.innerHTML = ''
      }
    }
  }, [slug])

  return <div id="giscus-container" />
}
```

## Phase 8: 다크 모드 및 UI 개선 (1주)

### 8.1 다크 모드 구현

#### 8.1.1 테마 컨텍스트
```typescript
// src/contexts/ThemeContext.tsx
'use client'

import { createContext, useContext, useEffect, useState } from 'react'

type Theme = 'light' | 'dark'

interface ThemeContextType {
  theme: Theme
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined)

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')

  useEffect(() => {
    const savedTheme = localStorage.getItem('theme') as Theme
    if (savedTheme) {
      setTheme(savedTheme)
    } else if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
      setTheme('dark')
    }
  }, [])

  useEffect(() => {
    document.documentElement.classList.toggle('dark', theme === 'dark')
    localStorage.setItem('theme', theme)
  }, [theme])

  const toggleTheme = () => {
    setTheme(theme === 'light' ? 'dark' : 'light')
  }

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export const useTheme = () => {
  const context = useContext(ThemeContext)
  if (context === undefined) {
    throw new Error('useTheme must be used within a ThemeProvider')
  }
  return context
}
```

## Phase 9: 최종 배포 및 최적화 (1주)

### 9.1 성능 최적화

#### 9.1.1 번들 분석
```bash
# 번들 분석 도구 설치
npm install -D @next/bundle-analyzer

# 분석 실행
npm run build
npm run analyze
```

#### 9.1.2 캐싱 전략
```typescript
// src/lib/notion.ts
export const getDatabase = async () => {
  // ISR을 위한 캐싱 설정
  const response = await notion.databases.query({
    database_id: process.env.NOTION_DATABASE_ID!,
    // ... 기존 설정
  })
  
  return response.results
}

// 페이지에서 사용
export const revalidate = 3600 // 1시간마다 재검증
```

### 9.2 최종 테스트 및 배포

#### 9.2.1 테스트 체크리스트
- [ ] 모든 페이지 로딩 확인
- [ ] 반응형 디자인 테스트
- [ ] SEO 메타데이터 확인
- [ ] 성능 점수 확인 (Lighthouse)
- [ ] 다크 모드 동작 확인
- [ ] 댓글 시스템 동작 확인

#### 9.2.2 최종 배포
```bash
# 프로덕션 빌드
npm run build

# 배포
vercel --prod
```

## 📅 개발 일정 요약

| Phase | 기간 | 주요 작업 |
|-------|------|-----------|
| Phase 1 | 1-2주 | 프로젝트 초기 설정, 기본 구조 |
| Phase 2 | 1-2주 | Notion API 연동, 데이터 페칭 |
| Phase 3 | 1주 | 기본 UI 컴포넌트 구현 |
| Phase 4 | 1주 | 첫 번째 배포 |
| Phase 5 | 2-3주 | 고급 기능 구현 |
| Phase 6 | 1주 | SEO 최적화, 성능 개선 |
| Phase 7 | 1주 | 댓글 시스템 구현 |
| Phase 8 | 1주 | 다크 모드, UI 개선 |
| Phase 9 | 1주 | 최종 배포 및 최적화 |

**총 예상 기간**: 10-12주

## 🎯 각 Phase별 성공 기준

- **Phase 1**: Next.js 프로젝트가 정상 실행되고 기본 구조가 완성됨
- **Phase 2**: Notion API에서 데이터를 정상적으로 가져올 수 있음
- **Phase 3**: 기본적인 블로그 목록과 상세 페이지가 표시됨
- **Phase 4**: 첫 번째 배포가 성공하고 기본 기능이 동작함
- **Phase 5**: 검색, 마크다운 렌더링, 목차 등 고급 기능이 구현됨
- **Phase 6**: SEO 점수가 90점 이상이고 성능이 최적화됨
- **Phase 7**: 댓글 시스템이 정상적으로 동작함
- **Phase 8**: 다크 모드가 완벽하게 동작하고 UI가 개선됨
- **Phase 9**: 최종 배포가 성공하고 모든 기능이 안정적으로 동작함

---

**문서 버전**: 1.0  
**작성일**: 2024년  
**작성자**: 개발팀 