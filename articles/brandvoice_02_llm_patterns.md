---
title: "ブランドボイスをLLMに学習させる実装パターン — Next.js + Claude API"
emoji: "🎙️"
type: "tech"
topics: ["NextJS", "Claude", "LLM", "TypeScript", "プロンプトエンジニアリング"]
published: true
---

## はじめに

「AIに文章を生成させると、どれも同じ感じになってしまう」

この問題の解決策が**ブランドボイスのLLM化**です。自社・自身の文体・トーン・用語・価値観をLLMに学習（プロンプトエンジニアリング）させることで、どんなコンテンツを生成しても「らしさ」が出るようになります。

本記事では、Next.js + Claude API を使ったブランドボイス実装パターンを解説します。

## ブランドボイスとは

ブランドボイスとは、企業やクリエイターが対外的に発信するコンテンツに一貫して流れる「声の個性」です。

- **トーン**: 丁寧/カジュアル、専門的/親しみやすい
- **スタイル**: 文章の長さ、句読点の使い方、語尾の傾向
- **語彙**: 使うべきワード・避けるべきワード
- **価値観**: コンテンツに込める思想・スタンス

これをLLMに正確に伝えることで、SNS投稿・メール・ブログ記事など、あらゆるコンテンツに一貫性が生まれます。

## 実装アーキテクチャ

```
┌─────────────────────────────────────────┐
│           Next.js App Router            │
│                                         │
│  ┌──────────────┐  ┌─────────────────┐ │
│  │ Brand Voice  │  │   Content       │ │
│  │ Editor       │→ │   Generator     │ │
│  └──────────────┘  └────────┬────────┘ │
│                             │           │
│  ┌──────────────────────────▼────────┐  │
│  │         /api/generate (Route)     │  │
│  └──────────────────────────┬────────┘  │
│                             │           │
└─────────────────────────────┼───────────┘
                              │
                    ┌─────────▼─────────┐
                    │   Claude API      │
                    └───────────────────┘
```

## Step 1: ブランドボイスのプロンプトテンプレートを設計する

ブランドボイスをLLMに伝えるには、**システムプロンプト**に組み込みます。

```typescript
// lib/brandVoice.ts
export type BrandVoice = {
  tone: string          // 例: "カジュアルで親しみやすく、専門用語を避ける"
  style: string         // 例: "短文中心、体言止めを多用、句読点は少なめ"
  vocabulary: {
    use: string[]       // 使うべき語句
    avoid: string[]     // 避けるべき語句
  }
  values: string        // 例: "ユーザーの成長を応援するスタンスを常に持つ"
  examples: string[]    // 実際の文章サンプル（3〜5件）
}

export function buildSystemPrompt(voice: BrandVoice): string {
  return `
あなたはブランドボイスに従ってコンテンツを生成するAIアシスタントです。

## トーン
${voice.tone}

## スタイル
${voice.style}

## 使うべき語句
${voice.vocabulary.use.join('、')}

## 避けるべき語句
${voice.vocabulary.avoid.join('、')}

## 価値観・スタンス
${voice.values}

## 文章サンプル（このスタイルを参考にすること）
${voice.examples.map((e, i) => `${i + 1}. ${e}`).join('\n')}

上記のブランドボイスに厳密に従って、依頼されたコンテンツを生成してください。
  `.trim()
}
```

## Step 2: APIルートでLLMを呼び出す

```typescript
// app/api/generate/route.ts
import Anthropic from '@anthropic-ai/sdk'
import { buildSystemPrompt } from '@/lib/brandVoice'
import type { BrandVoice } from '@/lib/brandVoice'

const client = new Anthropic()

export async function POST(req: Request) {
  const { prompt, brandVoice }: { prompt: string; brandVoice: BrandVoice }
    = await req.json()

  const systemPrompt = buildSystemPrompt(brandVoice)

  const message = await client.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    system: systemPrompt,
    messages: [{ role: 'user', content: prompt }],
  })

  const content = message.content[0]
  if (content.type !== 'text') {
    return Response.json({ error: 'Unexpected response type' }, { status: 500 })
  }

  return Response.json({ text: content.text })
}
```

## Step 3: ブランドボイスの保存と管理

ブランドボイスの設定は、ユーザーごとにデータベースに保存します。

```typescript
// lib/db/brandVoice.ts（Supabaseを使った例）
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)

export async function saveBrandVoice(userId: string, voice: BrandVoice) {
  const { error } = await supabase
    .from('brand_voices')
    .upsert({ user_id: userId, ...voice, updated_at: new Date().toISOString() })
  if (error) throw error
}

export async function getBrandVoice(userId: string): Promise<BrandVoice | null> {
  const { data, error } = await supabase
    .from('brand_voices')
    .select('*')
    .eq('user_id', userId)
    .single()
  if (error) return null
  return data as BrandVoice
}
```

## Step 4: Few-shot Examples の活用

ブランドボイス精度を上げる最も効果的な方法は、**実際の文章サンプルをfew-shotとして与える**ことです。

```typescript
export type ContentExample = {
  input: string   // 生成指示
  output: string  // 実際に使った文章
}

export function buildSystemPromptWithExamples(
  voice: BrandVoice,
  examples: ContentExample[]
): string {
  const examplesSection = examples.map(ex => `
---
指示: ${ex.input}
出力: ${ex.output}
---`).join('\n')

  return `${buildSystemPrompt(voice)}

## 具体的な入出力例
${examplesSection}

上記の例を参考に、同じスタイルで生成してください。`
}
```

## よくある落とし穴

### 1. プロンプトが曖昧すぎる

❌ 「フレンドリーなトーンで」
✅ 「文末は「〜ですよ」「〜ですね」を使い、敬語は丁寧語（〜です/〜ます）に限定する。タメ口NG」

具体性が命です。LLMは曖昧な指示を独自解釈します。

### 2. サンプル不足

最低でも3〜5件のサンプルを用意してください。サンプルが多いほど精度が上がります。

### 3. ネガティブリストを忘れる

「〇〇を使わない」という制約を入れることで、生成の精度が大幅に向上します。

## 実際に試してみる

今回解説したブランドボイス実装を体験できる **Brand Voice**（ https://brand-voice-sns.vercel.app ）を公開しています。SNS投稿・ブログ記事・メールをブランドボイスに合わせて自動生成できます。

## まとめ

- ブランドボイスはシステムプロンプトに構造化して組み込む
- Few-shot examplesを使うと精度が大幅向上
- ネガティブリスト（使わないワード）も必ず設定する
- Supabaseなどでユーザーごとの設定を永続化する

ブランドボイスのLLM化は、コンテンツ制作の生産性を大幅に上げながら一貫性も保てる強力な手法です。
