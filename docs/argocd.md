## Practical use of ArgoCD

ArgoCDはHelm Chartが公開されていることもあり、使うだけなら非常に簡単です。しかし、プロダクション環境で使うためには事前に様々な要素を考慮する必要があります。ここでは、以下の3点に注目してArgoCDを実用的に使うためにはどうすればいいのかを検討したいと思います。

- ArgoCDのベストプラクティスは？
- ArgoCDと他のCDツールの比較
- ベストプラクティスを適用できないケースはあるか？
- 実運用ではどんな要素を検討する必要があるか？

### Best practices of ArgoCD
まず、ArgoCD のベストプラクティスを確認していきたいと思います。

#### App of Apps Pattern 
ArgoCD では Application という Custom Resource (以下、CR)を用いてデプロイ対象のアプリケーションを宣言的に管理することができます。Application は ArgoCD に監視対象のアプリケーションのパスやデプロイ先の namespace などを認識させるためのリソースです。つまり、ArgoCD を使う環境で新たなアプリケーションを追加するためには以下の2つの作業が必要になります。以下では、簡単のためにアプリケーションは Deployment リソースとして定義されていることとします。

- 新しいアプリケーションの Deployment マニフェスト(`deployment.yaml`)を書く + GitHub に置く
- Application マニフェスト(`application.yaml`)を書く + GitHub に置く
- Application マニフェストをデプロイする(`kubectl apply -f application.yaml`)

ここで問題なのは「新しく作成するアプリケーションの Application リソースは手動でクラスタにデプロイする必要がある」ということです。アプリケーションを追加する度にコマンドラインで `kubectl apply -f <hoge-application>.yaml` を叩くのは GitOps 的にあまり良くないと思います。というのも、GitOps においては Git で管理されているマニフェストが唯一の信頼できる情報源であり、クラスタにデプロイされているリソースとGitで管理されているマニフェストは可能な限り一致していることが望ましいからです。`application.yaml`自体をGitで管理すること自体はもちろん可能ですが、担当者が `kubectl apply -f application.yaml` を打ち忘れたり、打ち込んだとしてもタイポしたりすると、Gitで管理されているマニフェストとクラスタ内のリソースに差異が発生してしまいます。

少し長くなりましたが、理想とする体験としては「`application.yaml`を書いてGithubのmainブランチにマージしたら、クラスタに`deployment.yaml`で定義した Deployment リソースがデプロイされた」です。これを実現するためには「Application リソースを管理する Application リソースを作成する」ことが必要であり、ArgoCD ではこれを App of Apps Pattern と呼んでいます。具体的には、`apps`のようなディレクトリを作成し、そこで Application マニフェストを管理します。新しいアプリケーションをデプロイするためには、以下の2つの作業を行います。

- 新しいアプリケーションの Deployment マニフェスト(`deployment.yaml`)を書く + GitHub に置く
- Application マニフェスト(`application.yaml`)を書く + GitHub の `apps` ディレクトリに置く

これにより、手動でコマンドを叩く必要がなくなり、GitHub で管理されているマニフェストとクラスタ内リソースの乖離が発生するリスクを小さくすることが出来ました。また、直接的なインパクトは小さいものの、オペレーションコストを減らすこともできています。

### Comparison of ArgoCD with other CD tools

WIP

### Are there cases where best practices cannot be applied?

WIP

### What factors need to be considered in production environments?