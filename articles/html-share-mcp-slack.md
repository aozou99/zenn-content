---
title: "HTML社内共有サービスにおけるMCP認証とSlackプレビューの設計"
emoji: "🔐"
type: "tech"
topics: ["mcp", "oauth", "slack", "aws", "auth0"]
published: false
---

:::message
本記事は [KODANSHAtech のブログ](https://kodansha.tech/ja/blogs/html-share) に掲載した記事の再録です。
:::

エンジニアの青木です。最近、社内でHTML共有ツール「HTMLShare」を作りました。提案したその日のうちに承認が出て、3営業日後には本番デプロイまでこぎ着けました。

「社内でHTMLを共有するツール」というアイデアや、実際に作ってみたという記事は、ここ最近あちこちで見かけるようになってきましたよね。AIがHTMLをサッと生成してくれるようになったので、それを社内に手軽に共有したいというニーズは、どの会社でも同時多発的に生まれているのだと思います。

ただ作っただけの話を書いてもあまり面白くないと思ったので、この記事では、自分がこだわった **MCP連携とSlackのリッチプレビュー** の2点に絞って書きます。同じようなツールを作ろうとしている方の参考になれば幸いです。

## きっかけ

Xのタイムラインに、「情報まとめにMarkdownをやめてHTMLを使っている。AnthropicもHTMLに移行している」という趣旨のポストが流れてきました（Anthropicも [Claude Code での HTML 活用についての記事](https://claude.com/blog/using-claude-code-the-unreasonable-effectiveness-of-html) を出しています）。これが、ちょうど自分が感じていたことと重なっていました。

最近、PoCのコンセプトを説明する資料をMarkdownで書くことに限界を感じていました。図や動画を埋め込みたい、インタラクティブにしたい、レイアウトを細かく調整したい——そのたびにMarkdownの制約に引っかかります。HTMLならすべてできますし、しかも今はAIが一瞬で書いてくれます。

ただ問題がありました。AIが生成したHTMLをどうやって社内に共有するか、です。HTMLファイルをそのまま送ると、相手はダウンロードして開く必要がありますし、画像や動画を参照しているHTMLだとファイル一式をZIPにまとめないと正しく表示されません。GitHub Pagesでホスティングする手も考えましたが、ちょっとしたHTMLを1枚共有したいだけにしては大げさです。結局、社内ドメインのGoogleアカウントでログインできる人だけがアクセスできて、認証付きURLでサッと渡せる——そんな社内ツールが欲しいと思い、提案してみたところ当日にGoサインが出ました。

そして提案した当日のうちにローカルで期待通り動くMVPを完成させ、翌日にデモを見せました。その場でAWSアカウント・Auth0テナント・Google Consoleのプロジェクトをそれぞれ管理者から払い出してもらえることになり、土日を挟んで次の営業日に本番デプロイしました。提案から3営業日という短期間でデプロイまで持っていけたのは、Claude Code/CodexによるAIコーディング支援をフル活用できたからです。インフラのAWS CDKのコードやAuth0のセットアップまわりは普段ならまとまった時間がかかる部分ですが、AIと壁打ちしながら進めることで大幅にスピードアップでき、こうした立ち上げが一気にやりやすくなったのを実感しました。

## 作ったもの

**HTMLShare** という名前の社内ツールで、社内メンバーがHTMLドキュメントを認証付きURLで共有できます。HTMLをアップロードすると共有用のURLが発行され、そのURLをSlackに貼るとリッチプレビューが表示されます。また、Claude CodeなどのMCPクライアントから直接アップロードすることもできます。

全体としてはAWS + Auth0で構成しており、CloudFrontを単一の入口として、React SPA・API・HTML配信・MCP・Slack連携をパスで振り分けています。

![HTMLShare のシステムアーキテクチャ図](/images/html-share/system-architecture.png)
*HTMLShare のシステムアーキテクチャ*

| レイヤー | 技術 |
| --- | --- |
| Frontend | React 18 + Vite + Auth0 |
| Backend | Hono on Lambda (API Gateway HTTP API) |
| 認証 | Auth0 + Google OAuth（社内アカウント限定） |
| DB | DynamoDB |
| ストレージ | S3 (private) + CloudFront OAC |
| IaC | AWS CDK |

ここまでは、おそらく他の社内HTML共有ツールとそれほど大きくは変わりません。HTMLをホスティングして認証付きURLを配る、というコア機能は共通です。ここから先が、自分がこだわったMCPとSlackの話になります。

## こだわり① MCP認証は「誰が共有したか」と「繋ぎやすさ」を両立させる

最近のこの手のツールは、たいていMCP（Model Context Protocol）に対応しています。Claude CodeやCodexのようなAIクライアントから直接アップロードできると便利だからです。HTMLShareも当然MCPに対応させたのですが、悩んだのが **MCPの認証をどう設計するか** でした。そもそもMCPの認証にどんな方法があるのかをよく知らなかったので、手探りでいくつか試してみることになりました。結果的にこの部分は一度作り直しており、その試行錯誤がそのまま「MCPの認証にはどんな選択肢があるか」の知見になったので、順を追って書きます。

### MCP認証の選択肢を整理する

MCPサーバーの認証方式は、大きく3つに整理できます。

| 方式 | 仕組み | 所感 |
| --- | --- | --- |
| APIキー / 静的トークン | 発行したトークンをMCPの設定ファイルに貼る | 手軽。ただし発行・失効を自前で管理する必要があり、トークンの持ち主と実際の操作者を結びつけにくい |
| OAuth + 固定クライアント | IdP（Auth0など）に事前にクライアントを1つ作り、その Client ID をユーザーに配って使い回す | ブラウザログインで本人認証できる。ただし Client ID の手渡しが必要で、対応できるMCPクライアントが限られる |
| **OAuth broker + 動的登録（DCR）** | **サーバー自身が authorization server になり、MCPクライアントが自動でクライアント登録する** | **ユーザーは何も貼らない。NotionやFigmaのMCPのように「接続→ブラウザ→完了」で繋がる** |

### 最初は「固定クライアント」方式だった

最初に作ったのは、表の真ん中の **OAuth + 固定クライアント** 方式でした。Auth0に Native アプリ（PKCE対応のpublic client）を1つ作り、MCPクライアントには「認可サーバーは Auth0 だよ」とメタデータで案内する構成です（このメタデータの仕組みは後ほど詳しく説明します）。Claude Codeなら、次のように Client ID とコールバックポートを渡せば動きます。

```bash
claude mcp add --transport http htmlshare https://htmlshare.example.com/mcp \
  --client-id <Auth0 Native アプリの Client ID> \
  --callback-port 8976
```

このときの認証の流れを図にすると、こうなります。**MCPクライアントが認可サーバーとして Auth0 をそのまま相手にしている** のがポイントです（Googleログインはブラウザのリダイレクトで行き来します）。

![旧方式（OAuth + 固定クライアント）の認証シーケンス図](/images/html-share/mcp-auth-fixed-client.png)
*旧方式：MCPクライアントが Auth0 を認可サーバーとして直接相手にする*

これはこれで動いていたのですが、使ってみて2つの不満が出てきました。

ひとつは、**Client ID をユーザーに配らないといけない** こと。MCPサーバーのURLとは別に「このClient IDを `--client-id` に渡してね」という案内が必要で、セットアップ手順が増えます。もうひとつは、**対応できるクライアントがClaude Codeにほぼ限られる** こと。`--client-id` や `--callback-port` を明示的に渡せるクライアントは限られますし、Codexなどは `http://127.0.0.1:<port>/callback/<id>` のような動的なループバックURLをコールバックに使うため、それらをAuth0側に事前登録しておくこともできません。

要するに、この方式は「Claude Codeでだけ、ちょっと手間をかければ動く」という状態でした。NotionやFigmaのMCPを使ったときの、URLを入れたらブラウザが開いてログインするだけ、という体験とは程遠かったのです。

### OAuth broker + 動的登録（DCR）に作り直す

そこで、表のいちばん下の **OAuth broker + 動的登録（Dynamic Client Registration / DCR）** 方式に作り直しました。

「OAuth broker」という言葉に馴染みがないかもしれないので、先に説明しておきます。OAuth broker とは、**クライアントから見ると「認可サーバー」として振る舞いながら、実際の認証は裏にいる本物のIdP（ここでは Auth0）に丸投げする仲介役** のことです。クライアントとAuth0の間に立って、OAuthのやり取りを取り次ぎます。旅行代理店をイメージすると分かりやすいかもしれません。利用者（クライアント）は代理店（broker）に希望を伝えるだけで、代理店が裏で航空会社やホテル（Auth0 / Google）とやり取りし、チケット（トークン）を持って帰ってくる。利用者が航空会社と直接やり取りする必要はありません。

HTMLShareでは、この broker を `/oauth/register`・`/oauth/authorize`・`/oauth/token` といったエンドポイント群として自前で実装しました。実際の認証フローを図にすると、こうなります。

![新方式（OAuth broker + 動的登録）の認証シーケンス図](/images/html-share/mcp-auth-broker-dcr.png)
*新方式：MCPクライアントは HTMLShare を認可サーバーとして扱い、broker が Auth0 との橋渡しをする*

旧方式と見比べると、MCPクライアントが **認可サーバーとして扱う相手が Auth0 から HTMLShare に変わっている** のが分かります。ブラウザはどちらの方式でもリダイレクトの途中で Auth0 や Google に行き来しますが、クライアントが「登録」や「トークン取得」のために相手にするのは HTMLShare だけになりました。`/oauth/register`（DCR）に対応したことで、MCPクライアントは自分のコールバックURLを実行時に自己登録し、Client IDを受け取れます。ユーザーがClient IDを手で渡す必要はなくなりました。Codexのような動的ループバックURLも、HTMLShare broker側で登録・照合するので問題ありません。Auth0には HTMLShare broker の固定コールバックURLを1つ登録しておくだけで済みます。

これで、登録コマンドはURLを渡すだけになりました。

```bash
# Client ID も callback port も指定しない
claude mcp add --transport http htmlshare https://htmlshare.example.com/mcp
# → 初回呼び出し時にブラウザが開いて Google ログイン → 完了
```

そして大事なのは、**繋ぎやすくしても「誰が共有したか」は分かるまま** ということです。brokerはあくまでAuth0への認可コードフローを仲介しているだけなので、最終的な認証は今までどおりGoogleアカウントで行われます。MCPクライアントからの共有も、WebUIからの共有とまったく同じく **そのユーザー本人の名義** で記録され、一覧・更新・削除もすべて本人だけが操作できます。AIに「このHTMLを共有して」と頼んでも、裏側ではちゃんと自分の名前で履歴が残る、という状態です。

ここで効いているのが **Auth0の存在** です。2つの図のどちらでも、Googleログイン連携・ユーザー情報（メールアドレスや名前）の取得・JWTの発行と署名・トークンの更新といった「認証そのもの」はすべてAuth0が引き受けてくれています（社内アカウントだけに絞る制限自体は、Auth0ではなくGoogle側を社内向けアプリとして設定することで効かせています）。おかげでHTMLShare側は、パスワードやGoogleとのOAuth連携を自前で実装する必要がありません。新方式で自作したのは、あくまで **MCP特有の作法（DCR・動的コールバック・メタデータ公開）をAuth0との間で橋渡しする薄い層** だけです。「認証の難しくて危険な部分はAuth0に任せ、MCPに繋ぐための薄い変換だけを自分で書く」——この役割分担が、短期間で安全に作れた一番の理由でした。

整理すると、APIキー方式は手軽だけれど誰が共有したのか分かりにくく、固定クライアント方式は共有した人は分かるけれど繋ぎにくい。DCR対応のOAuth brokerは、その両方——「誰が共有したか」を残しつつ、クライアントを選ばず繋がる——を満たせる、というのが今回たどり着いた結論でした。OAuth brokerの実装自体は仕様（OAuth 2.1 / [RFC 7591](https://www.rfc-editor.org/rfc/rfc7591)）を読みながら進めましたが、ここもClaude Code/Codexが大きく助けてくれた部分です。

### MCPクライアントはどうやってDCRに気づくのか

ここで、少しだけ仕組みを掘り下げてみます。さきほど「メタデータ公開」と書きましたが、そもそもMCPクライアントは、どうやって "このサーバーは動的登録に対応している" と気づくのでしょうか。URLを渡すだけで繋がる以上、どこかにその情報があるはずです。自分も実装しながら同じ疑問を持ちました。

答えは **メタデータに書いてある**、しかも探す場所は規約で決まっている、です。図の①「メタデータ取得」は、実際には次のように段階的にたどっていく流れになっています。

まず、クライアントが認証なしで `/mcp` を叩くと、サーバーは `401` と一緒に `WWW-Authenticate` ヘッダーで「メタデータはここにあるよ」と教えます（[RFC 9728](https://www.rfc-editor.org/rfc/rfc9728)）。

```text
GET /mcp（トークンなし）
→ 401 WWW-Authenticate: Bearer realm="htmlshare",
     resource_metadata="https://htmlshare.example.com/.well-known/oauth-protected-resource"
```

クライアントはそのURL（保護リソースメタデータ）を取得し、`authorization_servers` を読んで「認可サーバーは誰か」を知ります。続けてその認可サーバーのメタデータ（`/.well-known/oauth-authorization-server`）を取得すると、こんなJSONが返ってきます。ここに **`registration_endpoint` というフィールドがあれば、それがDCR対応の印** です。

```json
{
  "issuer": "https://htmlshare.example.com",
  "authorization_endpoint": ".../oauth/authorize",
  "token_endpoint": ".../oauth/token",
  "registration_endpoint": ".../oauth/register",
  "code_challenge_methods_supported": ["S256"]
}
```

クライアントはこの `registration_endpoint` を見つけたら、そこへ自分のコールバックURLをPOSTしてClient IDを動的に受け取ります（[RFC 7591](https://www.rfc-editor.org/rfc/rfc7591)）。逆にこのフィールドが無ければ「DCRはできない」と判断し、旧方式のように事前のClient IDが必要になります。

ここでのポイントは、**メタデータを探すパス（`/.well-known/oauth-protected-resource`、`/.well-known/oauth-authorization-server`）が規約で固定されている** ことです（それぞれ [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728) / [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414)）。だからクライアントは、サーバー固有の設定を事前に知らなくても「決まった場所」を見にいくだけで、認可サーバーの場所もDCRの可否も自動で判別できます。HTMLShare側は、このメタデータで「認可サーバーは自分自身」「`registration_endpoint` はここ」と宣言しているだけです。

そして、ここまでの「`/mcp` を叩く → 401 → メタデータをたどる」という一連の流れは、実は旧方式（固定クライアント）でもまったく同じでした。違うのは、たどり着いたメタデータの中身——認可サーバーが Auth0 自身か HTMLShare 自身か、そして `registration_endpoint` があるか（＝DCRできるか）——だけです。旧方式から新方式への切り替えも、突き詰めれば **メタデータの中身を入れ替えただけ** だったのです。

## こだわり② Slackのリッチプレビューは「安全な方法」で出す

MCPの次にこだわったのが、Slackのリッチプレビューです。共有URLをSlackに貼ったときに、タイトル・説明文・サムネイル付きのリッチなプレビューを出したい、と最初から思っていました。理由はふたつあって、ひとつは単純に、URLだけよりも中身が見えたほうが受け取る側に親切だから。もうひとつは社内ブランディングで、プレビューにOGP画像が出れば「HTMLShare というツールがあるんだ」という認知が社内に広まりやすいと考えたからです。ただ **認証コンテンツのプレビューをどう実現するか** は、実装方法によってセキュリティ特性が大きく変わります。

### 最初の実装：Slackbot を見分けて OGP を返す

最初に作ったのは、Slackのクローラーを見分けてプレビュー用のHTMLを返す方式でした。SlackはURLのプレビューを生成するとき、`Slackbot` というUser-Agentを名乗ってページを取得しに来ます。そこで、Lambda@Edgeでこのアクセスを検出したら認証をバイパスし、バックエンドがOGPメタタグだけのHTMLを返すようにしていました。実装も比較的単純で、これで一応Slack上にプレビューは出せていました。

### 気づいた脆弱性

ただ、リリース前に見直していて、この方式には無視できない穴があることに気づきました。**User-Agentは誰でも簡単に偽装できるため**、`Slackbot` を名乗るリクエストを投げるだけで、認証をバイパスする経路を通って、認証済みコンテンツのタイトルや説明文が誰にでも取得できてしまいます。そもそも「認証を迂回するルートを自分で用意してしまっている」こと自体が筋の悪い設計でした。せっかく社内ドメインのログインで守っているはずなのに、URLさえ知っていればログインしていない第三者でも中身を読めてしまう——これでは認証をかけている意味がありません。

### 他の手段を探して App Unfurling にたどり着く

そこで「認証を一切バイパスせずに、Slackにだけプレビューを渡す方法はないか」とSlackのAPIを調べ直し、たどり着いたのが [**App Unfurling**](https://docs.slack.dev/messaging/unfurling-links-in-messages) という仕組みでした。ワークスペースにアプリをインストールしておくと、そのドメインのURLがメッセージに貼られたとき、Slackが `link_shared` イベントをWebhookで送ってきます。アプリはそのイベントを受け取り、`chat.unfurl` APIでプレビュー内容を返します。最初の方式とは逆に、こちらは **Slackから送られてくるリクエストを検証する** 構造になっているのがポイントです。

![Slack App Unfurling のイベント受信からプレビュー展開までの7ステップ](/images/html-share/slack-unfurl-flow.png)
*Slack App Unfurling のイベント受信からプレビュー展開までの流れ。署名検証と許可ワークスペース判定は /slack/events 受信時、プレビュー本体は chat.unfurl で送信する*

鍵になるのは、Slackがすべてのリクエストにタイムスタンプ付きのHMAC署名を付けてくることです。これを検証することで、正規のSlackから来たリクエストかどうかを、User-Agentとは比べ物にならない信頼度で確認できます。さらに許可したworkspace IDからのイベントのみ処理するようにしており、そもそもHTML本体は一切渡さず、タイトル・説明文・OGP画像のURLだけをSlackに返しています。最初の方式が「認証を外して中身を返す」ものだったのに対し、この方式は「認証は外さず、信頼できる相手にだけメタ情報を渡す」ものになっています。

```typescript
export function buildSlackUnfurl(item, url, deliveryOrigin) {
  // HTML 本体は一切渡さず、メタ情報だけを返す
  return {
    color: '#4f46e5',
    title: item.title,
    title_link: url,
    text: item.description || 'HTMLShare - 社内HTML共有ツール',
    image_url: `${deliveryOrigin}/ogp/${item.shareId}`,
  }
}
```

Slack App Unfurlingのセットアップは、OGPを返すだけの方式に比べると少し手間がかかります（Slack Appの作成、イベント登録、ドメイン設定など）。それでも、認証コンテンツのSlackプレビューを安全に実装したいなら、User-Agent判定より先にこの方法を検討する価値があると思います。

### OGP画像はサーバーサイドで自動生成する

リッチプレビューに使うOGP画像は、サーバーサイドで自動生成しています。[satori](https://github.com/vercel/satori) でJSXライクなオブジェクトからSVGを生成し、[resvg-wasm](https://github.com/yisibl/resvg-js) でPNGに変換するという構成です。Reactが不要でLambda上でも動くのがちょうどよかったです。日本語の禁則処理・文字詰めには、社内で開発している tsume というライブラリを使っており、長いタイトルでもきれいに折り返されます（この日本語の文字詰めについては [KODANSHAtech のポスト](https://x.com/KODANSHAtech/status/2065661097760453041) でも紹介しています）。

タイトルと説明文は「ユーザーが明示入力した値 → HTMLの `<title>` / `<meta name="description">` → Amazon BedrockのClaude Haiku によるAI自動生成」という優先順位で補完します。AIによる生成は最後の手段という位置付けですが、この優先順位のおかげで、タイトルや説明文を書いていないHTMLでも情報の欠けたプレビューになりにくい——リッチなプレビューを保ちやすい造りにできたと思っています。また、アップロードした人のGoogleアカウントのアイコンと表示名もOGP画像に含めているので、Slackのプレビューから「誰が共有したか」がひと目でわかります。ここでも「誰が共有したか」が一貫して見える設計にしています。

実際に共有URLをSlackに貼ると、こんなふうにリッチなプレビューが表示されます。タイトル・説明文・自動生成したOGP画像（共有者のアイコン入り）が、認証コンテンツの中身を渡すことなく出せています。

![Slackに共有URLを貼ったときに表示されるリッチプレビューの例](/images/html-share/slack-preview.png)
*共有URLをSlackに貼ったときのリッチプレビュー*

## おわりに

MCPはOAuth brokerと動的登録（DCR）で、誰が共有したかを残したまま「URLを入れるだけで繋がる」体験にする。SlackはUser-Agent判定ではなく署名検証付きのApp Unfurlingで、認証コンテンツを漏らさずにプレビューを出す。どちらも「社内ツールだからまあいいか」で済ませず、外部とつながる部分こそ、安全性と使い勝手の両方をきちんと設計する、という方針で一貫させたつもりです。

そして個人的に大きな収穫だと感じているのが、**認証まわりの知見が溜まったこと** です。Auth0を土台にしたMCP向けOAuth brokerの組み方や、認証コンテンツを外部サービスへ安全に渡す方法は、次にまた別の社内ツールを作るときの認証基盤としてそのまま活かせるはずです。特にMCPの認証は選択肢が複数あって悩みやすいところなので、同じようなツールを作ろうとしている方の参考になれば幸いです。
