## Practical use of RBAC
RBAC(Role Based Access Control) は Kubernetes クラスタを安全に保つ上で非常に重要な役割を果たします。しかし、個人が趣味で Kubernetes を触っているだけでは RBAC のような機能を「ちゃんと」使う動機はあまりありません。かといって、実際の業務で Kubernetes や RBAC について触れる機会があるかどうかも人によってマチマチです。そこで、頭の中でできる限り具体的な開発組織を想定し、そこで RBAC をどのように設計・適用するかを考えてみます。一種の思考トレーニングのような側面もありますが、本気で考えてみると意外と理解が深まるのではないか、という予測のもと行う取り組みです。

## Development Team Structure
- Infrastructure Team
- Developer Team A
- Developer Team B
- Developer Team C
- SRE Team

## Requirement
まず、以下のようなざっくりとした要件を定義します。

- Infrastructure Team のメンバーは原則 Kubernetes の全てのリソースにアクセスできる
- Infrastructure Team のインターンメンバーは Production、Staging のリソースを変更できない
- Developer Team のメンバーは原則自身が所属する Team のリソースしか変更できない
- Developer Team のインターンメンバーは Production、Staging のリソースを変更できない
- Developer Team 間で権限の付与を(Infrastructure Teamを経由せずに)簡単に行いたい
- 権限の付与は常にGitHub上でのレビューを通過しないと実行できない


## Let's Thinking 
まず、デフォルトで用意されている権限でどこまで対応できそうか考えてみます。Infrastructure Team のメンバーはクラスタ内のすべてのリソースにアクセスできる必要がありますが、これはデフォルトの `cluster-admin` にかなり近いのではないでしょうか。

