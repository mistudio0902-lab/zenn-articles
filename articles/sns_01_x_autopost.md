---
title: "Claude CodeでX(Twitter)自動投稿システムを作る方法【コード生成から運用まで】"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "ai", "python", "自動化", "副業"]
published: true
---

## 3行でわかる本記事

- **対象**: プログラミング未経験からX APIの自動化に取り組みたい方
- **実現内容**: Claude Codeと無料ツール(tweepy + PM2)で、朝昼夜3回の定時投稿を完全自動化
- **所要時間**: X API申請～システム完成まで約1.5時間

---

## Claude Codeで始めるX自動投稿システム構築

毎日のX(Twitter)投稿は、継続が難しい。特に本業がある人にとって、定期的に投稿を続けるのは実装面での負担が大きいです。

プログラミング経験がなくても、Claude Codeに「X自動投稿システムを作ってほしい」と指示するだけで、必要なコード一式が生成されます。本記事では、その実装プロセスとコード例をすべて公開します。

## システムの全体構成

完成するシステムは以下の構造です：

```
【朝7:00】PM2がpost.pyを実行
  ↓ テンプレートを順番に選択
  ↓ X APIで投稿
  ↓ state.jsonで状態を更新

【昼12:00】【夜21:00】同じ処理を繰り返す
```

使用するツールはすべて無料です：

| ツール | 役割 | ライセンス |
|---|---|---|
| tweepy | X APIラッパーライブラリ | 無料 |
| PM2 | プロセス管理・スケジューラー | 無料 |
| Python 3.12 | 実行環境 | 無料 |

X APIの無料プランは月1,500ツイート。1日3投稿×30日=90投稿なので問題なく収まります。

## X Developer Portalでアプリケーションを登録

自動化する前に、X APIへのアクセス権を取得します。この作業だけは手動で行います。

1. [https://developer.twitter.com](https://developer.twitter.com) にアクセス
2. 「Sign up for Free Account」から登録
3. 使用目的を英語で記入（例：「Personal use for automated posting」）
4. アプリを作成し、以下の4つの認証情報をメモします

```
API Key (Consumer Key)         : xxxxxx
API Key Secret (Consumer Secret) : xxxxxx
Access Token                    : xxxxxx-xxxxxx
Access Token Secret             : xxxxxx
```

**重要**: デフォルトは「Read only」権限です。投稿するには「User authentication settings」から「Read and Write」に変更してください。この設定忘れが最も一般的な401エラーの原因です。

## Claude Codeへの最初の指示：プロジェクト構成

Claude Codeを起動し、以下を指示します：

```
X(Twitter)の自動投稿システムを作ってほしい。
要件:
- tweepyでX API v2を使う
- 朝7時・昼12時・夜21時の1日3回投稿
- テンプレートをローテーションで回す（毎回違う内容）
- PM2でスケジューリング
- 投稿が403で弾かれたときのフォールバック処理
- .envでAPIキーを管理
- 投稿ログを記録

まずプロジェクト構成を提案してください。
```

Claude Codeが提案する構成：

```
x-autopost/
├── post.py               # メインスクリプト
├── templates.py          # 投稿テンプレート36個
├── ecosystem.config.js   # PM2スケジューラー設定
├── state.json            # ローテーション状態（自動生成）
├── post.log              # 投稿ログ（自動生成）
├── .env                  # APIキー（Gitignoreに入れる）
└── .gitignore
```

## 投稿テンプレートの実装

テンプレートはアカウントの成長に直結するため、初期版をClaude Codeに生成させてから、自分で微調整するのがおすすめです。

Claude Codeへの指示：

```
12星座占いアカウント用の投稿テンプレートを作ってください。
- 朝: 今日の運勢（星座ごと、3星座ずつ4セット）
- 昼: 開運アクション・豆知識系
- 夜: 癒し・おやすみメッセージ
- 各12パターン、合計36テンプレート
- {date_tag}で日付を自動挿入
- エンゲージメント向上のためCTAを含める
```

Claude Codeが生成するテンプレートファイル：

```python
# templates.py
from datetime import datetime

def get_date_tag():
    """日付タグを「4月6日(日)」形式で生成"""
    d = datetime.now()
    weekdays = ["月", "火", "水", "木", "金", "土", "日"]
    return f"{d.month}月{d.day}日({weekdays[d.weekday()]})"

MORNING_TEMPLATES = [
    """\
【今日の運勢｜牡羊座〜双子座】
{date_tag}

🐏 牡羊座: 行動力が高まる日。新しい挑戦に向いている
♉ 牡牛座: 金運に恵まれる。買い物や投資の判断時
♊ 双子座: コミュニケーションが活発。人脈が広がる

あなたの星座は？コメントで教えてください✨
""",
    # ... 残り11パターン
]

NOON_TEMPLATES = [
    """\
【開運アクション】{date_tag}

✅ 朝日を浴びながら深呼吸3回
✅ 財布の中を整理する
✅ 周囲に「ありがとう」を言う

小さな習慣が運気を変えます🌟
""",
    # ... 残り11パターン
]

NIGHT_TEMPLATES = [
    """\
【今夜の星座メッセージ】{date_tag}

今日も頑張ったあなたへ。

星はいつもあなたを見ています。
明日もきっと良い日になる。

おやすみなさい🌙
""",
    # ... 残り11パターン
]
```

## メイン投稿スクリプト：3段階フォールバック付き

投稿時に403エラーが発生することは珍しくありません。このスクリプトは3段階のフォールバック処理で、投稿成功率を最大化します。

Claude Codeへの指示：

```
X APIへの投稿スクリプトを作ってください。
要件:
- tweepyでX API v2を使う
- 403 Forbiddenが出たらハッシュタグを除去して再投稿
- それでもダメならURLとCTAも除去
- 3段階のフォールバックで最大限投稿成功を目指す
- 投稿ログをpost.logに記録（ISO形式の日時 + slot + tweet_id）
```

生成されるメインスクリプト：

```python
#!/usr/bin/env python3
# post.py
import tweepy
import re
import sys
import os
import json
from pathlib import Path
from datetime import datetime
from dotenv import load_dotenv
from templates import (
    MORNING_TEMPLATES, NOON_TEMPLATES, NIGHT_TEMPLATES, get_date_tag
)

load_dotenv()

STATE_FILE = Path("state.json")
LOG_FILE = Path("post.log")

def load_state() -> dict:
    """ローテーション状態を読み込み"""
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text(encoding="utf-8"))
    return {"morning_idx": 0, "noon_idx": 0, "night_idx": 0}

def save_state(state: dict):
    """ローテーション状態を保存"""
    STATE_FILE.write_text(json.dumps(state, ensure_ascii=False, indent=2))

def get_next_template(slot: str) -> str:
    """次のテンプレートを取得してインデックスを進める"""
    state = load_state()
    
    templates = {
        "morning": MORNING_TEMPLATES,
        "noon": NOON_TEMPLATES,
        "night": NIGHT_TEMPLATES,
    }[slot]
    
    idx = state[f"{slot}_idx"] % len(templates)
    state[f"{slot}_idx"] = idx + 1
    save_state(state)
    
    return templates[idx].format(date_tag=get_date_tag())

def post_tweet(text: str) -> str:
    """
    3段階フォールバック付きで投稿
    Lv1: テンプレートそのままで投稿
    Lv2: ハッシュタグを除去（スパムフィルタ対策）
    Lv3: URLとCTA句も除去（コアテキストのみ）
    """
    client = tweepy.Client(
        consumer_key=os.getenv("X_API_KEY"),
        consumer_secret=os.getenv("X_API_KEY_SECRET"),
        access_token=os.getenv("X_ACCESS_TOKEN"),
        access_token_secret=os.getenv("X_ACCESS_TOKEN_SECRET"),
    )
    
    # Lv1: そのまま投稿
    try:
        response = client.create_tweet(text=text)
        return response.data["id"]
    except tweepy.errors.Forbidden:
        print("⚠️ Lv1失敗: Forbidden。ハッシュタグを除去します")
    
    # Lv2: ハッシュタグ除去
    text_no_hashtag = re.sub(r'\s*#\S+', '', text).strip()
    try:
        response = client.create_tweet(text=text_no_hashtag)
        return response.data["id"]
    except tweepy.errors.Forbidden:
        print("⚠️ Lv2失敗: URL・CTAを除去します")
    
    # Lv3: URL・CTA除去
    text_core = re.sub(r'https?://\S+', '', text_no_hashtag).strip()
    text_core = re.sub(r'（[^）]+）$', '', text_core).strip()
    
    response = client.create_tweet(text=text_core)
    return response.data["id"]

def log_post(slot: str, tweet_id: str):
    """投稿ログを記録"""
    with open(LOG_FILE, "a", encoding="utf-8") as f:
        timestamp = datetime.now().isoformat(timespec='seconds')
        f.write(f"{timestamp} [{slot}] {tweet_id}\n")

def main():
    if len(sys.argv) < 2:
        print("Usage: python post.py [morning|noon|night]")
        sys.exit(1)
    
    slot = sys.argv[1]
    text = get_next_template(slot)
    
    print(f"=== {slot.upper()} POST ===")
    print(text)
    print()
    
    try:
        tweet_id = post_tweet(text)
        print(f"✅ 投稿成功: {tweet_id}")
        log_post(slot, tweet_id)
    except Exception as e:
        print(f"❌ 投稿失敗: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

## PM2によるスケジューリング設定

Claude Codeへの指示：

```
PM2でpost.pyを定時実行する設定を作ってください。
- 朝7時、昼12時、夜21時の3回
- PC再起動後も自動起動
- ecosystem.config.js形式で出力
```

生成される設定ファイル：

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: "x-autopost-morning",
      script: "post.py",
      args: "morning",
      interpreter: "python",
      cron_restart: "0 7 * * *",
      autorestart: false,
      error_file: "logs/morning.err.log",
      out_file: "logs/morning.out.log",
    },
    {
      name: "x-autopost-noon",
      script: "post.py",
      args: "noon",
      interpreter: "python",
      cron_restart: "0 12 * * *",
      autorestart: false,
      error_file: "logs/noon.err.log",
      out_file: "logs/noon.out.log",
    },
    {
      name: "x-autopost-night",
      script: "post.py",
      args: "night",
      interpreter: "python",
      cron_restart: "0 21 * * *",
      autorestart: false,
      error_file: "logs/night.err.log",
      out_file: "logs/night.out.log",
    },
  ],
};
```

## セットアップと起動

依存パッケージをインストールします：

```bash
pip install tweepy python-dotenv
npm install -g pm2
```

`.env` ファイルを作成（Step 1で取得したAPIキーを入力）：

```
X_API_KEY=your_api_key_here
X_API_KEY_SECRET=your_api_key_secret_here
X_ACCESS_TOKEN=your_access_token_here
X_ACCESS_TOKEN_SECRET=your_access_token_secret_here
```

`.gitignore` を作成（APIキーをコミットしない）：

```
.env
.DS_Store
*.pyc
__pycache__/
node_modules/
logs/
state.json
post.log
```

PM2を起動します：

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

動作確認：

```bash
pm2 list                    # プロセス一覧を表示
pm2 logs x-autopost-morning # 朝の投稿ログを確認
pm2 logs --err              # エラーログを確認
```

## 動作テスト

実際に投稿できるか事前に確認しましょう：

```bash
python post.py morning      # 朝のテンプレートで即座にテスト投稿
python post.py noon         # 昼のテンプレートで即座にテスト投稿
python post.py night        # 夜のテンプレートで即座にテスト投稿
```

post.log を確認して、tweet_id が記録されていれば成功です：

```bash
cat post.log
# 出力例
# 2026-04-06T07:00:15 [morning] 1234567890123456789
```

## よくあるトラブルと対処

### 401 Unauthorized

認証情報が不正です。確認項目：

1. Developer Portalで「User authentication settings」が「Read and Write」になっているか
2. `.env` のAPIキーが正確にコピーされているか
3. キーの先頭・末尾に空白がないか

対処：キーを再生成して `.env` に入力し直してください。

### 403 Forbidden

投稿内容がスパム判定されています。本スクリプトは自動でフォールバック処理を実行します。

テンプレートのバリエーション不足が続く場合は、重複するテンプレートを削減し、日付タグの使用を確認してください（毎投稿、異なる日付が埋め込まれるべき）。

### 429 Too Many Requests

APIレート制限に達しました。無料プランの月1,500ツイート制限を超えていないか確認してください。

```bash
# post.log の行数を数える
wc -l post.log
```

月間投稿数が1,500を超える場合は、有料プランへのアップグレードが必要です。

### PM2がpythonを見つけられない

`ecosystem.config.js` の `interpreter` を修正します：

```javascript
// フルパスで指定する場合
interpreter: "/usr/bin/python3",  // macOS/Linux
interpreter: "C:\\Python312\\python.exe",  // Windows
```

pythonのパスを確認：

```bash
which python3  # macOS/Linux
where python   # Windows
```

## まとめ

Claude Codeを活用すれば、プログラミング経験がなくても、本番運用レベルの自動化システムを構築できます。

| 項目 | 自力でコーディング | Claude Code活用 |
|---|---|---|
| 必要知識 | Python・tweepy・PM2・API | 不要 |
| 実装時間 | 4〜6時間 | 約30分 |
| テスト・デバッグ | 自分で全対応 | ログをClaude Codeに貼って指示 |
| フォールバック設計 | 仕様理解が必須 | Claude Codeが提案 |

本システムは完全自動で、毎日7時・12時・21時に投稿されます。その間、あなたの作業は不要です。

---

## さらに詳しい情報はnoteで

本記事では要点を抜粋しました。プロンプト全文・テンプレート36個の詳細・運用ノウハウ・拡張例は note でまとめています。

https://note.com/mi_autolab
