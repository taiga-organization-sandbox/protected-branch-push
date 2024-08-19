# package.jsonのversion更新 & タグ付けプッシュを自動化


## 前提条件

* mainブランチに対して以下のrulesetsを設定している。
    * `Require a pull request before merging`で`Required approvals`を設定。
    * `Block force pushes`を設定。
* 無料枠のorganizationでrulesetsを利用するため、repositoryはpublicにしている。

## Githubにおけるトークンの選択肢とGithub Actionsで権限範囲

| トークンの種類 | 権限範囲 | 説明 | 
| --- | --- | --- |
| 🙅 ①`GITHUB_TOKEN` | ***保護されたブランチへのプッシュ不可*** | Github Actionsがデフォルトで持っているシークレット。できる内容に制限あり。 | 
| 🔺 ②Personal Access Token (PAT)(個人のアカウントで作成) | ***直接Push不可。PRのReview・Approveの自動化によりMainにMerge(Push)できるが、使用に際しては課題あり。*** | ・作業メンバーが自身のアカウントでTokenを発行し、対象のリポジトリのsecretsに設定することで使用可能。(scopesはrepo/ workflows) <br> ・❌トークンの有効期限: 無期限にしないと有効切れの度に入れ替えが必要。<br>・❌特定のユーザーへの紐づき: 対象のユーザーがorganizationから外れると、再度設定が必要になる。<br>❌セキュリティリスク: PATはその発行したユーザーの持つ全てのリポジトリや組織、エンタープライズに対するアクセス権限を持つ。スコープの限定などの対処もできるが、アクセス範囲を制限することはできない。そのため、外部への漏洩や不正アクセスなどのリスクが発生する。(←完全に理解できていない) <br> *ユースケースは`.github/workflows/pat-auto-pr.yml`を参照。| 
| 🔺 ③Personal Access Token (PAT)(マシンユーザーを作成し設定) | ***②と同じ*** | ②と同じ <br> ・❌ユーザ課金の発生: マシンユーザにも料金が発生する。 | 
| 🙆 ④GitHub Apps | ***①〜③の課題を解決した上で、直接Pushが可能 (rulesetsのbypassに設定)*** | ・👌トークンの有効期限/ セキュリティリスク: Github Actions上で発行され、短時間（一般的には1時間）で有効期限が切れる。つまり、トークンの手動管理や有効期限の心配が不要だし、万が一トークンが漏洩した場合でも迅速に無効化され、不正利用のリスクが低減される。<br> ・👌ユーザ課金の発生: ユーザではないため、課金されない。<br>・🔺Github Appをbypassに含めることの是非。(直接Pushするためにはbypass listに含める必要がある) <br> *ユースケースは`.github/workflows/github-app-push.yml`を参照。|

## Github Appの利用

### Github App作成
* `Developer settings`=>`Github Apps`
* App nameはユニークなもの
* Homepage URLはprivateの利用の場合は何でも良いので、今回はorganization(`taiga-organization-sandbox`)のURLで設定
* WebhookはActiveのチェックを外す。(←完全に理解できていないが、チェックを外すのが適切なプラクティスらしい)
    * 不要な通知を避ける: 初期段階ではどのイベントが必要か明確でないため、不必要な通知を防ぐ。
    * セキュリティとパフォーマンスの考慮: 不適切な設定が攻撃リスクやパフォーマンス低下を引き起こす可能性があるため。
* Permissionsの設定
    * Contents=>Read and write (これによってbypass list に当該Appを設定できる)
    * Metadata=>Read-only (mandatory)
* Appの公開設定はデフォルトでprivateとなるが、作成後、念の為確認する。(Advancedから確認できる)

### Repositoryのrulesets変更
* `bypass list`に作成したGithub Appを設定。

### Github Actionsでの利用
* `Developer settings`=>`Github Apps`=>`Install App`でインストール
    * `Only select repositories`で対象のリポジトリを選択
* Private Keys作成: 
    * `Developer settings`=>`Github Apps`=>`General`=>`Generate Private Keys`=>ダウンロード
* Repositoryのvariables/secretsの設定
    * App ID: `Developer settings`=>`Github Apps`=>`General`=>`App ID`=>variables or secretsで設定
    * Private Keys: ダウンロードしたpemファイルにアクセス=> 文字列をSecretsに設定
* Github ActionsでCheckoutする際にトークンを発行し、Pushのステップで当該トークンを使用。(詳細は、`.github/workflows/github-app-push.yml`を参照。)

## Github Actionsのパッケージ化 (再利用)

### 今回のケース
* 設定
    * 再利用するActionワークフローを格納したリポジトリを作成 ([`taiga-org-gh-actions-packages`](https://github.com/taiga-organization-sandbox/taiga-org-gh-actions-packages))
        * `.github/workflows/`にaction用のymlファイルを格納しても良いし、rootディレクトリに格納しても良い。
        * 呼び出し先で入力するversionの入力値、宣言するsecretsを受け取れるよう設定する。
    * 呼び出し元のymlファイル (`.github/workflows/github-app-push-reusable.yml`) で、Versionの入力値とGithub App用の`Private Keys`を渡すよう設定し、usesで再利用actionのpathを宣言。
        * `@xxx`でバージョンを指定することも可能。

## 参考文献
* [GitHub Apps overview(公式Doc)](https://docs.github.com/en/apps/overview)
* [GitHub Access Tokens explained](https://devopsjournal.io/blog/2022/01/03/GitHub-Tokens)
* [Create a GitHub App from a manifest](https://devopsjournal.io/blog/2021/12/27/GitHub-App-from-manifest)
* [mainブランチに対して直接pushを禁止する保護ルールを設定をする](https://zenn.dev/json_hardcoder/articles/f9b534377103a4)
* [GitHub Appとは？ 作りながら仕組みを理解する](https://zenn.dev/takamin55/articles/569875e8346948#github-apps%E3%81%A8%E3%81%AF)
* [そのマシンユーザー不要ですよ！GitHub Appsを使ってGitHub Actionsを利用しよう](https://zenn.dev/tatsuo48/articles/72c8939bbc6329#%E3%81%AF%E3%81%98%E3%82%81%E3%81%AB)
* [作って学ぶ GitHub Apps](https://note.com/teitei_tk/n/n5ad51f00a006)
* [peter-murray/workflow-application-token-action(Short Lived Tokenを発行するActions)](https://github.com/peter-murray/workflow-application-token-action)