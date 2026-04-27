---
title: "複数プラットフォームへの自動投稿を1つのコードで管理する"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "ai", "python", "自動化", "副業"]
published: true
---

## 複数SNS/プラットフォームへの自動投稿を Dispatcher パターンで一元管理する

**個人開発者向け** | **複数プラットフォーム（note・X・Instagram など）への投稿を1つのコードベースで管理** | **所要時間：30分～1時間**

---

## はじめに

個人プロダクト・副業の情報発信では、同じコンテンツを複数プラットフォームに配信することが珍しくありません。しかし、各プラットフォームの API 仕様・文字数制限・フォーマット要件が異なるため、平気で 10 倍の工数が必要になります。

本記事では、**Dispatcher パターン**を用いて、複数プラットフォームへの投稿を効率的に管理する設計を紹介します。新しいプラットフォームを追加する際も、最小限の変更で対応できる構造です。

---

## Dispatcher パターン：役割分離の設計

複数プラットフォームへの投稿を管理する際、以下の 2 つの責務を明確に分離することが重要です。

### Dispatcher の責務

**「どのプラットフォームに、どんなフォーマットで送るか」を決定する司令塔**

- 投稿対象プラットフォームの判別（環境変数で制御）
- コンテンツのフォーマット変換（プラットフォーム別の仕様適用）
- 投稿タイミング・スケジューリング管理
- リトライ戦略の統括

### PostPilot の責務

**「実際に各 API を叩く」という実行役**

- 各プラットフォームの API 呼び出し実装
- HTTP レスポンスの解析・バリデーション
- プラットフォーム固有のエラーハンドリング
- タイムアウト・接続エラー対応

この分離により、新しいプラットフォーム追加時は PostPilot に API 実装を足し、Dispatcher にフォーマット変換ロジックを追加するだけで対応できます。

---

## 環境変数による投稿制御の実装

ローカルテストと本番投稿を簡単に切り替えるため、環境変数で投稿対象を制御します。

```bash
# .env.local（テスト環境）
ENABLED_PLATFORMS=note
POST_IMMEDIATELY=false
RETRY_MAX_ATTEMPTS=3
RETRY_BACKOFF_MS=5000

# .env.production（本番環境）
ENABLED_PLATFORMS=note,x,instagram,hatena
POST_IMMEDIATELY=true
RETRY_MAX_ATTEMPTS=5
RETRY_BACKOFF_MS=8000
```

環境ごとに投稿対象を簡単に切り替えられるため、本番環境でのバグが減ります。

---

## Dispatcher の実装例

プラットフォーム仕様を TypeScript で定義し、Dispatcher が動的にフォーマット変換・投稿制御を行う実装です。

```typescript
// platformSpecs.ts
interface PlatformSpec {
  maxLength: number
  supportsMarkdown: boolean
  formatTransformer: (content: string) => string
  apiEndpoint: string
}

const platformSpecs: Record<string, PlatformSpec> = {
  note: {
    maxLength: 65536,
    supportsMarkdown: true,
    formatTransformer: (content) => content,
    apiEndpoint: 'https://note.com/api/v2/posts'
  },
  x: {
    maxLength: 280,
    supportsMarkdown: false,
    formatTransformer: (content) => 
      content
        .replace(/\n\n/g, '\n')
        .replace(/\n/g, ' ')
        .substring(0, 280),
    apiEndpoint: 'https://api.twitter.com/2/tweets'
  },
  instagram: {
    maxLength: 2200,
    supportsMarkdown: false,
    formatTransformer: (content) => 
      content + '\n\n#副業 #自動化',
    apiEndpoint: 'https://graph.instagram.com/v18.0/me/media'
  },
  hatena: {
    maxLength: 10000,
    supportsMarkdown: true,
    formatTransformer: (content) => `[はじめに]\n\n${content}`,
    apiEndpoint: 'https://blog.hatena.ne.jp/api/atom/entry'
  }
}

export function getPlatformSpec(platform: string): PlatformSpec {
  const spec = platformSpecs[platform]
  if (!spec) throw new Error(`Unknown platform: ${platform}`)
  return spec
}
```

---

## Dispatcher クラスの実装

```typescript
// dispatcher.ts
import { getPlatformSpec } from './platformSpecs'

interface PostResult {
  platform: string
  success: boolean
  postId?: string
  error?: string
}

export class Dispatcher {
  private enabledPlatforms: string[]
  private retryMaxAttempts: number
  private retryBackoffMs: number

  constructor() {
    const platformStr = process.env.ENABLED_PLATFORMS || 'note'
    this.enabledPlatforms = platformStr.split(',').map(p => p.trim())
    this.retryMaxAttempts = parseInt(process.env.RETRY_MAX_ATTEMPTS || '3')
    this.retryBackoffMs = parseInt(process.env.RETRY_BACKOFF_MS || '5000')
  }

  async publishToAllPlatforms(
    content: string,
    metadata?: Record<string, any>
  ): Promise<PostResult[]> {
    const results: PostResult[] = []

    for (const platform of this.enabledPlatforms) {
      try {
        const spec = getPlatformSpec(platform)
        const formattedContent = spec.formatTransformer(content)

        if (formattedContent.length > spec.maxLength) {
          throw new Error(
            `Content exceeds ${platform} limit: ${formattedContent.length} > ${spec.maxLength}`
          )
        }

        const result = await this.postWithRetry(
          platform,
          formattedContent,
          metadata
        )
        results.push(result)
      } catch (error) {
        results.push({
          platform,
          success: false,
          error: error instanceof Error ? error.message : String(error)
        })
      }
    }

    await this.notifyResults(results)
    return results
  }

  private async postWithRetry(
    platform: string,
    content: string,
    metadata?: Record<string, any>,
    attempt: number = 1
  ): Promise<PostResult> {
    try {
      const postPilot = new PostPilot(platform)
      const postId = await postPilot.post(content, metadata)
      
      return {
        platform,
        success: true,
        postId
      }
    } catch (error) {
      if (attempt >= this.retryMaxAttempts) {
        throw error
      }

      const backoff = this.retryBackoffMs * Math.pow(2, attempt - 1)
      console.log(
        `Retry ${platform} after ${backoff}ms (attempt ${attempt}/${this.retryMaxAttempts})`
      )
      
      await this.sleep(backoff)
      return this.postWithRetry(platform, content, metadata, attempt + 1)
    }
  }

  private async notifyResults(results: PostResult[]) {
    const failed = results.filter(r => !r.success)
    
    if (failed.length > 0) {
      console.error('Failed postings:', failed)
      // Slack・Discord などに通知
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms))
  }
}
```

---

## PostPilot（実行役）の実装

各プラットフォームの API 呼び出しを担当します。

```typescript
// postPilot.ts
interface PostPilotAdapter {
  post(content: string, metadata?: Record<string, any>): Promise<string>
}

class NoteAdapter implements PostPilotAdapter {
  async post(content: string): Promise<string> {
    const response = await fetch('https://note.com/api/v2/posts', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.NOTE_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        title: '自動投稿のテスト',
        content,
        status: 'published'
      })
    })

    if (!response.ok) {
      throw new Error(`Note API error: ${response.statusText}`)
    }

    const data = await response.json()
    return data.post_id
  }
}

class XAdapter implements PostPilotAdapter {
  async post(content: string): Promise<string> {
    const response = await fetch('https://api.twitter.com/2/tweets', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.X_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ text: content })
    })

    if (!response.ok) {
      throw new Error(`X API error: ${response.statusText}`)
    }

    const data = await response.json()
    return data.data.id
  }
}

export class PostPilot {
  private adapter: PostPilotAdapter

  constructor(platform: string) {
    switch (platform) {
      case 'note':
        this.adapter = new NoteAdapter()
        break
      case 'x':
        this.adapter = new XAdapter()
        break
      // 他のプラットフォーム...
      default:
        throw new Error(`Unsupported platform: ${platform}`)
    }
  }

  async post(content: string, metadata?: Record<string, any>): Promise<string> {
    return this.adapter.post(content, metadata)
  }
}
```

---

## 使用例

```typescript
// main.ts
import { Dispatcher } from './dispatcher'

const dispatcher = new Dispatcher()

const content = `
複数プラットフォームへの自動投稿は、適切な設計で大幅に効率化できます。
Dispatcher パターンを用いることで、新しいプラットフォーム追加も容易になります。
`

const results = await dispatcher.publishToAllPlatforms(content, {
  title: 'Dispatcher パターンの実装',
  tags: ['自動化', '設計パターン']
})

console.log('投稿結果:', results)
// 出力例:
// [
//   { platform: 'note', success: true, postId: 'abc123' },
//   { platform: 'x', success: true, postId: 'def456' },
//   { platform: 'instagram', success: false, error: '...' }
// ]
```

---

## エラーハンドリングとリトライ戦略

プラットフォーム API は一時的なエラー・レート制限に直面することがあります。指数バックオフを用いたリトライで信頼性を高めます。

```typescript
// リトライ戦略の詳細
private async postWithRetry(
  platform: string,
  content: string,
  metadata?: Record<string, any>,
  attempt: number = 1
): Promise<PostResult> {
  try {
    const postPilot = new PostPilot(platform)
    const postId = await postPilot.post(content, metadata)
    
    return { platform, success: true, postId }
  } catch (error) {
    const isRetryable = this.isRetryableError(error)
    
    if (!isRetryable || attempt >= this.retryMaxAttempts) {
      return {
        platform,
        success: false,
        error: error instanceof Error ? error.message : String(error)
      }
    }

    const backoff = this.retryBackoffMs * Math.pow(2, attempt - 1)
    await this.sleep(backoff)
    return this.postWithRetry(platform, content, metadata, attempt + 1)
  }
}

private isRetryableError(error: any): boolean {
  // 429: Rate Limit, 503: Service Unavailable, etc.
  return error.status === 429 || error.status === 503 || error.code === 'ECONNRESET'
}
```

---

## マルチプロダクト運用への拡張

複数の副業プロジェクトを運用する場合、環境変数を YAML 設定に昇格させると管理しやすくなります。

```yaml
# config/products.yaml
products:
  product-a:
    enabledPlatforms: [note, x, instagram]
    postImmediately: true
    retryMaxAttempts: 5
    
  product-b:
    enabledPlatforms: [note, hatena]
    postImmediately: false
    retryMaxAttempts: 3
```

Dispatcher をプロダクト単位でインスタンス化することで、プロダクト別の投稿戦略が実現できます。

---

## まとめ

Dispatcher パターンを用いた設計により：

- **新規プラットフォーム追加が容易** — PostPilot に API 実装を追加するだけ
- **テスト・本番の切り替えが簡単** — 環境変数で制御
- **エラー追跡が明確** — 各プラットフォームの結果を個別に管理
- **スケーリングに対応** — マルチプロダクト運用も可能

シンプルな設計ですが、個人開発者の情報発信を大きく効率化します。

---

## さらに詳しい情報はnoteで

本記事では要点を抜粋しました。完全な実装手順・エラーハンドリングの詳細・運用ノウハウ・監視・ダッシュボード構築については note でまとめています。

https://note.com/mi_autolab
