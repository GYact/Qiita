---
title: "Claude Code × Agent Skills | SKILL.mdの設計から自動発火・権限制御まで"
tags:
  - ClaudeCode
  - Anthropic
  - AIエージェント
  - 自動化
  - CLI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「また同じ手順書をチャットに貼り付けている」――この感覚は正しいです。

デプロイ手順、コミット規約、社内API設計ルール。同じコンテキストを毎回プロンプトに貼っていると、そのうち「毎回貼る」こと自体が仕事になります。Claude Code の Agent Skills は、この「毎回貼るコンテキスト」を `SKILL.md` という1ファイルに固定し、必要な時だけ Claude 自身に読み込ませる仕組みです。カスタムスラッシュコマンドと違い、**ユーザーが呼ばなくても Claude が description を見て自発的に発火できる**のが最大の違いで、ここが設計を誤ると「発火しない」「余計な時に発火する」の両方の事故につながります。

この記事では、最小構成から権限制御・自動発火制御・サブエージェント実行まで、実際に手を動かして検証できる手順で追っていきます。手元のプロジェクトでも project スキルを何本も運用していて、書き方を間違えると「発火してほしい時に発火しない」問題に必ず一度は当たるので、そこも踏まえて書きます。

## Step 1: 最小構成のSKILL.mdを作る

Agent Skills はディレクトリ単位です。プロジェクト直下に置くと、そのリポジトリの全セッションで有効になります。