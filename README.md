Concourse チュートリアル（日本語訳）
==================

このチュートリアルを用いて、https://concourse.ci の使い方とコンセプトをステップバイステップで学びましょう。

はじめに
---------------

Vagrant/Virtualboxをインストールします。

このチュートリアルを取得して、さぁ始めましょう。

```
git clone git@github.com:nitky/concourse-tutorial.git
cd concourse-tutorial
vagrant up
```

ブラウザで http://192.168.100.4:8080/ を開いてください。

[![initial](http://cl.ly/image/401b2z2B3w17/no-pipelines.png)](http://192.168.100.4:8080/)

使用しているOSに応じた`fly`インターフェースをダウンロードします。

![cli](http://cl.ly/image/1r462S1m1j1H/fly_cli.png)

ダウンロードしたら, 実行ファイル`fly` を`/usr/local/bin` や `~/bin`などの (`$PATH`)が通ったフォルダに入れてください。また、`fly`のファイルを実行可能にすることを忘れないでください。実行権限を与えるには、以下のコマンドを使用します。

```
sudo mkdir -p /usr/local/bin
sudo mv ~/Downloads/fly /usr/local/bin
sudo chmod 0755 /usr/local/bin/fly
```

Concourse のターゲット
----------------

いつも完全に同じ結果を得るために、`fly` はすべての`fly`リクエストに対してAPIのターゲットを定めることを要求します。

まず、`tutorial`という別名を与えます。（この名前は全てのチュートリアルのラッパースクリプトとして使用されています）

```
fly --target tutorial login  --concourse-url http://192.168.100.4:8080 sync

```

ここまでの操作によって、ローカルファイルの中に保存された Concourse APIのターゲットを見ることができます。


```
cat ~/.flyrc
```

APIや資格情報などの情報は以下のようなシンプルなYAMLファイルで表されます。

```yaml
targets:
  tutorial:
    api: http://192.168.100.4:8080
    username: ""
    password: ""
    cert: ""
```

`fly` コマンドを使うとき、我々は `fly -t tutorial`と打つことで、このConcourse APIをターゲットにできます。

> @alexsuraci: I promise you'll end up liking it more than having an implicit target state :) Makes reusing commands from shell history much less dangerous (rogue fly configure can be bad)

チュートリアル
---------

### 01 - Hello World task

```
cd 01_task_hello_world
fly -t tutorial execute -c task_hello_world.yml
```

上のコマンドを皮切りに、このようなログが出力されます。

```
Connecting to 192.168.100.4:8080 (192.168.100.4:8080)
-                    100% |*******************************| 10240   0:00:00 ETA
initializing with docker:///busybox
```

Concourseにある全てのタスクは(目的のプラットフォームに最善な形の)"コンテナ"で実行されます。`task_hello_world.yml`コンフィグレーションは、`docker:///busybox`に定義されたコンテナイメージを使用する`linux`プラットフォームの上で実行させることを示しています。

このコンテナの中で`echo hello world`コマンドを実行させましょう。


```yaml
---
platform: linux

image: docker:///busybox

run:
  path: echo
  args: [hello world]
```

出力の前の時点では、Docker イメージ `busybox`のダウンロードを行っています。
ダウンロードはただ一回実行するだけでよく、実行時にはいつも最新の`busybox` イメージがあるかどうかをチェックしてくれます。

さらに続けて行きましょう。

```
running echo hello world
hello world
succeeded
```

`image:` と `run:` を 違うタスクに変えて実行してみましょう。

```yaml
---
platform: linux

image: docker:///ubuntu#14.04

run:
  path: uname
  args: [-a]
```

このタスクファイルは便利に使えます。

```
$ fly -t tutorial execute -c task_ubuntu_uname.yml
Connecting to 192.168.100.4:8080 (192.168.100.4:8080)
-                    100% |*******************************| 10240   0:00:00 ETA
initializing with docker:///ubuntu#14.04
running uname -a
Linux mjgia714efl 3.13.0-49-generic #83-Ubuntu SMP Fri Apr 10 20:11:33 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
succeeded
```

コマンドを直接実行するよりも、Concourse タスクはシェルスクリプトのラッパーである`run:` で実行することが一般的なパターンです。

タスクとラッパースクリプトを複雑なパイプラインの中に押し込めていく際は、次のようなルールを考えてみましょう。

-	タスクファイルとラッパーシェルスクリプトに同じベースネームを与える

`01_task_hello_world` フォルダの中にある２つのファイルをみてください。

-	`task_show_uname.yml`
-	`task_show_uname.sh`

`fly`によってタスクファイルが直接実行されるとき、タスクの入力としてカレントフォルダがアップロードされます。これはラッパーシェルスクリプトが実行に利用できることを意味しています。

```
$ fly -t tutorial execute -c task_show_uname.yml
Connecting to 192.168.100.4:8080 (192.168.100.4:8080)
-                    100% |*******************************| 10240   0:00:00 ETA
initializing with docker:///busybox
running ./task_show_uname.sh
Linux mjgia714eg3 3.13.0-49-generic #83-Ubuntu SMP Fri Apr 10 20:11:33 UTC 2015 x86_64 GNU/Linux
succeeded
```
上記の出力である`running ./task_show_uname.sh`は、`task_show_uname.yml`タスクがラッパーシェルスクリプトにタスクの仕事を任せたことを示しています。

`task_show_uname.yml` タスクは次のようになっています。

```yaml
platform: linux
image: docker:///busybox

inputs:
- name: 01_task_hello_world
  path: .

run:
  path: ./task_show_uname.sh
```

ここで、新たに示されたコンセプト`inputs:`を説明します。

タスクがラッパースクリプトを実行するためには、ラッパースクリプトへアクセス方法を与えなければいけません。
同様に、タスクがデータファイルを処理するためには、データファイルへアクセス方法を与える必要があります。

Concourseにおいて、それらはタスクのための`inputs`になります。

`fly`によって直接タスクが実行させることで、`01_task_hello_world`の中にある私たちのホストマシンから実行された後、
現在のホストマシンのフォルダがConcourseにアップロードされ、`01_task_hello_world`と呼ばれる入力が利用可能になります。

そして入力に伴うジョブを見た後で、ジョブ内部のタスクへの入力を通じてタスクと出力は返されます。

上に示した`inputs`のスニペットについて考慮すると、

```yaml
inputs:
- name: 01_task_hello_world
  path: .
```

これらは次のことを言っています。

1.	私は`01_task_hello_world` と呼ばれる入力フォルダを受け取りたい。
2.	私はそれを `.` フォルダに置きたい。(`.`は実行されるとき、タスクのルートフォルダとみなされる)

デフォルトでは、`path:`に何もなかった場合、入力自身と同じ名前のフォルダに置き換えられます。

`inputs`のリストが与えられたとき、（同じフォルダの中にある）`task_show_uname.sh`スクリプトが実行タスクのルートフォルダの中で利用可能になります。

これは、次のような実行を許可します。

```yaml
run:
  path: ./task_show_uname.sh
```

### 02 - Hello World job

```
cd ../02_job_hello_world
fly set-pipeline -t tutorial -c pipeline.yml -p 02helloworld
fly unpause-pipeline -p 02helloworld
```

これらはコンコースのパイプライン（と変更点がないこと）を表示し確認を求めます。

```yaml
jobs:
  job job-hello-world has been added:
    name: job-hello-world
    public: true
    plan:
    - task: hello-world
      config:
        platform: linux
        image: docker:///busybox
        run:
          path: echo
          args:
          - hello world
```

`fly set-pipeline` (もしくは `fly sp`)を実行するたび、なにも変更がないことの確認を承諾するプロンプトが表示されます。

```
apply configuration? (y/n):
```

`y`を押してください。

次のようなメッセージが表示されるはずです:

```
pipeline created!
you can view your pipeline here: http://192.168.100.4:8080/pipelines/02helloworld
```

ブラウザに戻ってジョブを手動で始めましょう。`job-hello-world`をクリックした後、右上隅にある大きな`+`をクリックしてください。
ジョブが実行されます。

![job](http://cl.ly/image/3i2e0k0v3O2l/02-job-hello-world.gif)

左上隅の"Home"アイコンをクリックすると、パイプラインのステータスが表示されます。

### 03 - Tasks extracted into resources

前に示した`pipeline.yml`の中にあるコンフィグレーションによってジョブのタスクは簡単に繰り返すことができます。しまいには、すでに外へ出したリソースのジョブタスクを同じ場所に配置したくなるのかもしれません。

これは少し複雑な"hello world" タスクの例ですが、私たちが実行したいタスクが事前に示した"01 - Hello World task"からのものであると仮定しましょう。
それはgitリポジトリに保存されています。

`pipeline.yml`の中へ、チュートリアルのgitリポジトリにリソースとして加える。

```yaml
resources:
- name: resource-tutorial
  type: git
  source:
    uri: https://github.com/starkandwayne/concourse-tutorial.git
```

今、私たちはジョブをリソースとして消費することができます。アップデートしましょう。

```yaml
jobs:
- name: job-hello-world
  public: true
  plan:
  - get: resource-tutorial
  - task: hello-world
    file: resource-tutorial/01_task_hello_world/task_hello_world.yml
```


最初の`get`は`resource-tutorial`というリソースを得るために必要だ、と私たちの`plan:`で明示されています。
次に、タスクコンフィグレーションである`resource-tutorial`から`01_task_hello_world/task_hello_world.yml`のファイルを使用します。

`fly set-pipeline -t tutorial -c pipeline.yml -p 03_resource_job`を用いてパイプラインのアップデートを承認しましょう。

脚注: `fly` は楽にコマンドが打てるように短いエイリアスがある。`fly sp` が `fly set-pipeline`と省略できるように。

あるいは、チュートリアルによって事前に構築されたパイプラインを実行してみましょう。

```
cd ../03_resource_job
fly sp -t tutorial -c pipeline.yml -p 03_resource_job
fly unpause-pipeline -t tutorial -p 03_resource_job
```

![resource-job](http://cl.ly/image/271z3T322l25/03-resource-job.gif)

UIでジョブを手動実行した後、出力はこのようになります。

![job-task-from-wrapper](http://cl.ly/image/0Q3m223v2l3M/job-task-from-wrapper.png)

`job-hello-world` で定義されたビルドプランのジョブには二つのステップがあります。

一つ目のステップは、gitリポジトリからトレーニング教材とチュートリアルを取得することです。この"リソース"は`resource-tutorial`と呼ばれています。

このリソースはどんなジョブのビルドプランを含むタスクでも入力とすることができます。

二つ目のステップは、ユーザ定義のタスクを実行することです。UIの出力に表示された、タスク名`hello-world`が与えられます。タスク自身は
パイプラインの中で記述されません。その代わり、`resource-tutorial`の入力によって`01_task_hello_world/task_hello_world.yml`の中に記述されます。

パイプラインの外側に出して、YAMLファイルの中にタスクを記述することは利点と欠点がそれぞれあります。

利点は、作用するインプットリソースとマッチするようにタスクの振る舞いを修正できることが挙げられます。例えば、もし入力リソースがテスト付きのコードリポジトリであるならば、コストを実行するためにどのようなコードがリポジトリに必要なのかタスクファイルと同期を保ち続けることができます。

欠点は、`pipeline.yml`がどんなコマンドを実行するのかを正確に記述しないことです。ファイルから理解できることが減るということです。
`pipeline.yml`ファイルが長くなればなるほど、全てのYAMLを読んで理解することは困難になるかもしれません。

これらの選択をする場合、他のチームメンバの理解を考える必要があります。「このパイプラインは実際何をするんだ！」

ひとつのアイデアはタスクファイルの名前付けをどうするか考慮することです。例えば、ラッパースクリプトの実行内容によって名前付けを行うのも良いかもしれません。

その目的と振る舞いを説明する（長い）名前を使うことを考えましょう。

大切なことは`pipeline.yml`を読めるように作ることです。それはチーム/会社/プロジェクトの中で重要なオーケストレーションのツールになるでしょう。誰もがどのように、それが実際どう動作するのかを知っているベキです。

### 04 - Get job output in terminal

`job-hello-world` はgitレポジトリとタスクの実行で取って来られたリソースからのターミナル出力を持ちます。
また、ターミナルで`fly`を実行すると、このような出力を見ることができるでしょう。

```
fly -t tutorial watch -j 03_resource_job/job-hello-world
```

出力は次のようになるでしょう。

```
Cloning into '/tmp/build/get'...
e8c6632 Added trigger: true to autostart both jobs after update.
initializing with docker:///busybox
running echo hello world
hello world
succeeded
```

### 05 - Trigger a Job via the Concourse API

vagrant中のConcourseは `http://192.168.100.4:8080`で実行中のAPIがある。デフォルトで`fly`はこのエンドポイントをターゲットにします。

そのAPIを使って実行させることで、ジョブを起動することができる。例えば、`curl`:を用いて

```
curl http://192.168.100.4:8080/pipelines/03_resource_job/jobs/job-hello-world/builds -X POST
```

上のコマンドを実行するとき、`fly watch`を用いることで、上のコマンドターミナルの出力を見ることができる、

```
fly -t tutorial watch -j 03_resource_job/job-hello-world
```

### 06 - Triggering jobs - the `time` resource

"ソースは毎分チェックされるが、ビルドが実行されるべきとき、それを決定するための少しのインターバル（10秒）がある。時間リソースは、おおよそ周期的にビルドが実行されることを保証する必要がある。これらのことを用いて、例えば私たちはダメなところを見つけ出すために、インテグレーション／アクセプタンスのテストを継続的に実行する" - アレックス

最終的な結果は、`2m`のタイマーが毎度2分から3分の間に起動するということです。

### 20 - Available concourse resources

https://github.com/concourse?query=resource

-	[bosh-deployment-resource](https://github.com/concourse/bosh-deployment-resource) - パイプラインの一部として構成されたboshリリースのデプロイ
-	[semver-resource](https://github.com/concourse/semver-resource) - セマンティックバージョンの自動更新
-	[bosh-io-release-resource](https://github.com/concourse/bosh-io-release-resource) - bosh.io上のリリースバージョンの追跡
-	[s3-resource](https://github.com/concourse/s3-resource) - AWS S3 に関するConcourseのリソース
-	[git-resource](https://github.com/concourse/git-resource) - gitリポジトリの中のコミットの追跡
-	[bosh-io-stemcell-resource](https://github.com/concourse/bosh-io-stemcell-resource) - bosh.io上のstemcellのバージョンの追跡
-	[vagrant-cloud-resource](https://github.com/concourse/vagrant-cloud-resource) - manages boxes in vagrant cloud, by provider
providerによるvagrant cloudのボックスの管理
-	[docker-image-resource](https://github.com/concourse/docker-image-resource) - dockerイメージ用リソース
-	[archive-resource](https://github.com/concourse/archive-resource) - uriからのダウンロードとアーカイブ(今のところtgz)の展開
-	[github-release-resource](https://github.com/concourse/github-release-resource) - githubリリースのためのリソース
-	[tracker-resource](https://github.com/concourse/tracker-resource) - pivotal trackerのリソース出力
-	[time-resource](https://github.com/concourse/time-resource) - 定期的にトリガーを引くためのリソース
-	[cf-resource](https://github.com/concourse/cf-resource) - Cloud Foundryに関するConcourseのリソース

コンコースでどのようなリソースが利用可能かは、APIエンドポイント `/api/v1/workers`:で探し出すことができる。

```
$ curl -s http://192.168.100.4:8080/api/v1/workers | jq -r ".[0].resource_types[].type" | sort
archive
bosh-deployment
bosh-io-release
bosh-io-stemcell
cf
docker-image
git
github-release
s3
semver
time
tracker
vagrant-cloud
```
