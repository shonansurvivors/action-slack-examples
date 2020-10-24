## はじめに

GitHub ActionsでSlack通知を行う方法は様々あり、専用のActionも多数ありますが、[`8398a7/action-slack`](https://github.com/8398a7/action-slack) を使うとお手軽に色々な情報を表示できるのでおすすめです。

主に以下のことができます。

- GitHub上のリポジトリへのリンク表示
- PRへのリンク表示
- コミットへのリンク表示
- プッシュを行ったGitHubアカウント名の表示
- GitHub Actionsの処理結果へのリンク表示
- ブランチ名の表示
- 好きなメッセージを表示できる(裏技的な使い方になりますが)
- メンションを追加できる
- ジョブが失敗した時のみメンションする、といったこともできる

![all](https://user-images.githubusercontent.com/45457022/97071238-16a7f980-161a-11eb-9f6b-ed6a816b95ab.png)

本記事では、その使い方を紹介します。

## 事前準備

Slackアプリを作成し、発行したIncoming Webhook URLをGitHubのsecretsに設定しておいてください。

## 最も簡単な使い方

以下は最も簡単な使い方の例です。

`main`ブランチにPRがマージされるとデプロイが実行され、その結果をSlack通知する内容となっています。

(本記事ではデプロイ処理の中身は割愛し、`exit 0`するだけのステップとしています)

```yaml:.github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run deployment process
        run: exit 0

      - name: Notify deployment
        uses: 8398a7/action-slack@v3.8.0
        with:
          status: ${{ job.status }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always()
```

`main`ブランチにPRがマージされ、ジョブが正常終了すると以下の通知が来ます。

![basic_success](https://user-images.githubusercontent.com/45457022/97070014-9086b580-160f-11eb-9fc3-e0f438c9b402.png)

デフォルトで、リポジトリとコミットへのリンクが表示されます。

Slack通知を行うステップには、`if: always()`を付けているので、それよりも前のステップでエラーになってもSlack通知のステップは処理されます。

以下は、エラーが発生した場合の通知です。

![basic_failed](https://user-images.githubusercontent.com/45457022/97070062-30444380-1610-11eb-94b5-8c8e50051a6c.png)

## より多くの情報を表示する

より多くの情報を表示したい場合は、`with`に`fields`を追加してください。

以下は表示可能な全情報を指定した例です(なお、1つ1つ指定せず`all`と指定しても構いません)。

```yaml:.github/workflows/deploy.yml
      - name: Notify deployment
        uses: 8398a7/action-slack@v3.8.0
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,job,took,eventName,ref,workflow # 追加
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always()
```

この場合、以下のような通知が来ます。

![all](https://user-images.githubusercontent.com/45457022/97070238-d6447d80-1611-11eb-8766-27d937b6bae8.png)

それぞれの項目の意味は以下の通りです。

| Field | 内容 |
| ---- | ---- |
| repo | リポジトリ名とそのリンク |
| message | コミットメッセージとそのリンク |
| commit | コミットハッシュとそのリンク |
| author | プッシュ(マージ)した人 |
| action | GitHub Actionsの実行結果へのリンク |
| job | GitHub Actionsのジョブ名とそのリンク |
| took | ジョブの処理時間 |
| eventName | GitHub Actionsのトリガーとなったイベント名 |
| ref | Gitのリファレンス(`refs/heads/ブランチ名`など) |
| workflow | GitHub Actionsのワークフロー名とそのリンク |

こんなに情報があると見づらいので絞りたい、という人は必要なものだけを指定すればOKです。

## 好きなメッセージを表示する

Slack通知上の`8398a7@action-slack`と表示されている部分は、変更が可能です。

その場合は、`with`に`author_name`を追加してください。

(この`author_name`は、`with.fields`に指定する`author`とは別物ですので注意してください)

例えば、以下のように何の通知であるかを示すメッセージを入れると、よりわかりやすくなるかもしれません。

```yaml:.github/workflows/deploy.yml
      - name: Notify deployment
        uses: 8398a7/action-slack@v3.8.0
        with:
          status: ${{ job.status }}
          author_name: foobarサービスのデプロイ処理結果 # 追加
          fields: repo,message,commit,author,action,job,took,eventName,ref,workflow
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always()
```

![author_name](https://user-images.githubusercontent.com/45457022/97070709-810a6b00-1615-11eb-9237-35b855cbe771.png)

## ジョブの失敗時のみメンションする

ジョブが失敗した時、今回の例で言えばデプロイ処理が失敗した時に特定のSlackユーザーグループ(SREチームなど)へメンション付きで通知できると、デプロイ失敗のアラートとなるので便利です。

そうしたことを行いたい場合は、`mention`に通知先のグループID、`if_mention`にメンションするときの条件を指定します。

以下は、ジョブが失敗した時のみメンション付きで通知する例です。

```yaml:.github/workflows/deploy.yml
      - name: Notify deployment
        uses: 8398a7/action-slack@v3.8.0
        with:
          status: ${{ job.status }}
          author_name: foobarサービスのデプロイ処理結果
          fields: repo,message,commit,author,action,job,took,eventName,ref,workflow
          mention: 'subteam^ユーザーグループID' # 追加
          if_mention: failure # 追加
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always()
```

以下のようにジョブ失敗時はメンション付きで通知されます。

![mention](https://user-images.githubusercontent.com/45457022/97071150-37bc1a80-1619-11eb-80dd-35ed7d97ccf2.png)

なお、SlackのユーザーグループIDの調べ方ですが、適当なメンション付きの既存メッセージをブラウザ版のSlackで開き、さらにユーザーグループをクリックすると、アドレスバーに

```
https://app.slack.com/client/Txxxxxxxx/Cxxxxxxxx/user_groups/Sxxxxxxxx
```

のように、`/user_groups/ユーザーグループID`が表示されるので、そこを見るのが手っ取り早いかもしれません。

## 思いきりカスタマイズして使う

お手軽に有用な情報を表示できるのが`8398a7/action-slack`の良いところだと思いますが、表示内容を思いきりカスタマイズして使うこともできるようです。

その方法に関しては、公式の以下のページを参照ください。

- [Custom use case - action-slack](https://action-slack.netlify.app/usecase/02-custom)

## 終わりに

以上、`8398a7/action-slack`の紹介でした。

既存のGitHub Actionsのワークフローに、さくっとSlack通知を追加するのにはとても便利かと思います。

本記事が役立てば幸いです。

## 参考

- https://github.com/8398a7/action-slack
- [Document - action-slack](https://action-slack.netlify.app/)
