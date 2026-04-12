---
title: "AIがブランドボイスを学習してSNS投稿を自動生成するSaaSをClaude Codeで作った話"
emoji: "🎨"
type: "tech"
topics: ["ClaudeCode", "Gemini", "SaaS", "個人開発", "NextJS"]
published: true
---

# AIがブランドボイスを学習してSNS投稿を自動生成するSaaSをClaude Codeで作った話

個人開発でSNS運用ツール「BrandVoice AI」を作りました。

**URL:** https://brand-voice-sns.vercel.app/

## どんなツールか

美容サロン・飲食店・コンサルタントなど、中小企業のSNS担当者が抱える「毎回投稿文を考えるのがしんどい」を解決するツールです。

業種・ターゲット層・トーン（親しみやすい/プロフェッショナル/高級感など）を設定すると、AIがブランドボイスを学習。「今週末キャンペーンを告知したい」とトピックを入力するだけで、X・Instagram・Facebook向けに最適化された3パターンの投稿文を即座に生成します。

- 登録不要
- クレカ不要
- 60秒でスタート

## 技術スタック

- **フロントエンド/バックエンド**: Next.js 15 (App Router)
- **AI**: Gemini 2.5 Flash API
- **スタイリング**: Tailwind CSS
- **ホスティング**: Vercel
- **状態管理**: localStorage（DB不要でゼロコスト運用）

## 作るうえで一番悩んだこと：ブランドボイスの表現

「ブランドボイス」をAIにどう伝えるか、が一番難しかった。

単に「フレンドリーなトーンで書いて」だけでは再現性が低い。業種・ターゲット層・既存の投稿サンプルを組み合わせたプロンプト設計を何度も試行錯誤しました。

最終的には以下の情報をシステムプロンプトに埋め込む形に：

```
業種: 美容サロン・エステ
ターゲット: 30〜40代女性
トーン: 親しみやすい・カジュアル
ブランド説明: 地元密着のオーガニック系サロン
キーワード: リラクゼーション, 安心, 丁寧
```

これで一貫性のある投稿が出るようになりました。

## Gemini 2.5 Flashを選んだ理由

当初は `gemini-2.0-flash` を使っていましたが、リリース後に廃止されて `gemini-2.5-flash` に移行しました。

```typescript
const res = await fetch(
  `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${apiKey}`,
  { method: "POST", ... }
);
```

レスポンス速度が速く、API料金も安いのでフリープランのユーザーが大量に使っても赤字になりにくい設計です。

## Claude Codeで開発した感想

LP・Dashboard・Generate・APIルート全部含めて、実質1日で動くものができました。

特に助かったのは：
- Tailwind CSSのクラス自動補完
- API routeのエラーハンドリング自動追加
- 「白背景+Google色のデザイン」という指示だけで統一感のあるUIが出来上がる点

## 現在の状態と今後

- 無料プラン: 月10回生成（現在制限なし）
- Pro: ¥2,980/月（決済実装中）
- Business: ¥9,800/月（チーム対応）

ProductHuntへのローンチも予定しています。

使ってみた感想や「こんな機能が欲しい」などフィードバックあればコメントお願いします。

https://brand-voice-sns.vercel.app/
