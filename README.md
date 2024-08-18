# Github Actionsでpackage.jsonのversion更新 & タグ付けプッシュを自動化

## 前提条件
* mainブランチに対して以下のrulesetsを設定している。
    * 

## トークンの選択肢
A) 🙅`GITHUB_TOKEN`: ***保護されたブランチへのプッシュはできない。***
* Github Actionsがデフォルトで持っているシークレット。

B) 🔺Personal Access Token (PAT): ***直接Pushはできないが、PRを保護されたブランチへのプッシュはできない。***

* 作業メンバーが自身のアカウントで発行し、対象のリポジトリ
no expiration (not recommended?)
* scopes: all for repo/ workflowa