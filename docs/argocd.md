## Practical use of ArgoCD

### Disclamer
このドキュメントでは ArgoCD に関する様々な調査や検討を行います。確かなエビデンスに基づいているものは断定的に書くこともありますが、多くの場合は個人の見解や考えたことであり、むしろこれから議論していきたいトピックを特に扱う予定です。現時点で分からないことを明確にすることもこのドキュメントの1つの役割です。そのため、このドキュメントに書かれている内容を参考にした結果発生したいかなる問題についても、筆者は責任を負いません。

### Perspective
ArgoCDはHelm Chartが公開されていることもあり、使うだけなら非常に簡単です。しかし、プロダクション環境で使うためには事前に様々な要素を考慮する必要があります。ここでは、以下の4点に注目してArgoCDを実用的に使うためにはどうすればいいのかを検討したいと思います。

- ArgoCDのベストプラクティスは？
- ArgoCDと他のCDツールの比較
- ベストプラクティスを適用できないケースはあるか？
- 実運用ではどんな要素を検討する必要があるか？

### Best practices of ArgoCD
まず、ArgoCD のベストプラクティスを確認していきたいと思います。

#### Manage Argo CD Using Argo CD
ArgoCD は Kubernetes クラスタ上で動作しているため、やろうと思えば ArgoCD 自体を ArgoCD で管理することも可能です。もちろん ArgoCD を初めてクラスタにデプロイする時はコマンドを叩いて手動でデプロイする必要があります。具体的には、ArgoCD においてデプロイ対象となるアプリケーションを管理するための Application という Custom Resource を用いて、 ArgoCD のマニフェストの変更を監視させることで実現できます。実際のディレクトリ構成の一例として、以下のようなものが挙げられます（公式では kustomize を用いた例があるため、それをそのまま引用しています）。

```
argocd/
┣ manifests/
   ┣ base/
      ┣ argo-cd-certificate.yaml
      ...
   ┣ overlays/
      ┣ production/
        ┣ argo-cd-cm.yaml
        ┣ argocd-notifications-cm.yaml
       ... 
   ┣ kustomization.yaml
┣ argocd-self-application.yaml
```

使い始めだとイマイチ有難みが分からないかもしれませんが、例えば「新しいマイクロサービスを ArgoCD でデプロイできるようにしたい」という場合に役に立ちます。マイクロサービスアーキテクチャを採用している場合、マイクロサービスごとにリポジトリを分けていることが多いと思います。ArgoCD を ArgoCD で管理していない状態では、監視対象のリポジトリを追加するためには、以下の3つの方法があります。 

- `argo-cd-cm.yaml` を変更して `kubectl apply -f argo-cd-cm.yaml` を実行する
- ArgoCD のUIから直接リポジトリを登録する
- `argocd repo add` や `argocd app create --repo` を実行する

パッと見で分かる通り、UIを使うかコマンドラインを使うかどうかです。辛うじて最初の方法ではマニフェストで ArgoCD を管理していますが、`argo-cd-cm.yaml` が ArgoCD で管理されていない以上、GitOps のやり方からは外れています。ここで理想とする体験は「`argo-cd-cm.yaml`を変更してGitHubのmainブランチにマージしたら ArgoCD に新しいリポジトリが登録された」です。これを実現するためには ArgoCD 自体を1つの Application リソースとして管理する必要があります。具体的には、以下のような `argocd-application.yaml` を書いてクラスタにデプロイします。



#### App of Apps Pattern 
既に述べたように、ArgoCD では Application というリソースを用いてデプロイ対象のアプリケーションを宣言的に管理することができます。Application は ArgoCD に監視対象のアプリケーションのパスやデプロイ先の namespace などを認識させるためのリソースです。つまり、ArgoCD を使う環境で新たなアプリケーションを追加するためには以下の2つの作業が必要になります。以下では、簡単のためにアプリケーションは Deployment リソースとして定義されていると仮定します。

- 新しいアプリケーションの Deployment マニフェスト(`deployment.yaml`)を書く + GitHub に置く
- Application マニフェスト(`application.yaml`)を書く + GitHub に置く
- Application マニフェストをデプロイする(`kubectl apply -f application.yaml`)

ここで問題なのは「新しく作成するアプリケーションの Application リソースは手動でクラスタにデプロイする必要がある」ということです。アプリケーションを追加する度にコマンドラインで `kubectl apply -f <hoge-application>.yaml` を叩くのは GitOps 的にあまり良くないと思います。というのも、GitOps においては Git で管理されているマニフェストが唯一の信頼できる情報源であり、クラスタにデプロイされているリソースとGitで管理されているマニフェストは可能な限り一致していることが望ましいからです。`application.yaml`自体をGitで管理すること自体はもちろん可能ですが、担当者が `kubectl apply -f application.yaml` を打ち忘れたり、打ち込んだとしてもタイポしたりすると、Gitで管理されているマニフェストとクラスタ内のリソースに差異が発生してしまいます。

少し長くなりましたが、理想とする体験としては「`application.yaml`を書いてGithubのmainブランチにマージしたら、クラスタに`deployment.yaml`で定義した Deployment リソースがデプロイされた」です。これを実現するためには「Application リソースを管理する Application リソースを作成する」ことが必要であり、ArgoCD ではこれを App of Apps Pattern と呼んでいます。具体的には、`apps`のようなディレクトリを作成し、そこで Application マニフェストを管理します。

```
apps/
　┣ application-a.yaml
　┣ application-b.yaml
　┣ application-c.yaml <-- Add new application manifest
```

新しいアプリケーションをデプロイするためには、以下の2つの作業を行います。

- 新しいアプリケーションの Deployment マニフェスト(`deployment.yaml`)を書く + GitHub に置く
- Application マニフェスト(`application.yaml`)を書く + GitHub の `apps` ディレクトリに置く

これにより、手動でコマンドを叩く必要がなくなり、GitHub で管理されているマニフェストとクラスタ内リソースの乖離が発生するリスクを小さくすることが出来ました。また、直接的なインパクトは小さいものの、オペレーションコストを減らすこともできています。


### Comparison of ArgoCD with other CD tools

WIP

### Are there cases where best practices cannot be applied?

WIP

### What factors need to be considered in production environments?