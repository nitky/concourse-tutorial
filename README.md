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

いつも完全に同じ結果を得るために、`fly` はすべての`fly`リクエストに対する特別なターゲットAPIを要求します。

まず、`tutorial`という別名を与えます。（`tutorial`は全てのチュートリアルのラッパースクリプトとして使用されています）

```
fly --target tutorial login  --concourse-url http://192.168.100.4:8080 sync

```

ここまでの操作によって、ローカルファイルの中にある Concourse API の保存されたターゲットを見ることができます。


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

出力は以下のようになる。

```
Connecting to 192.168.100.4:8080 (192.168.100.4:8080)
-                    100% |*******************************| 10240   0:00:00 ETA
initializing with docker:///busybox
```

Concourseにある全てのタスクは"コンテナ"で実行される(ターゲットプラットフォームに最善の形で)。`task_hello_world.yml`コンフィグレーションは、`docker:///busybox`によって定義されたコンテナイメージを用いた`linux`プラットフォームの実行を示しています。

このコンテナの中で`echo hello world`コマンドを実行させましょう。


```yaml
---
platform: linux

image: docker:///busybox

run:
  path: echo
  args: [hello world]
```

出力がある前の時点では、Docker イメージ `busybox`のダウンロードを行っている。
ダウンロードはただ一回実行するだけでよく、実行時にはいつも最新の`busybox` イメージがあるかをチェックしてくれる。

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

直接的なコマンドの呼び出しよりも、Concourse タスクの`run:` にやらせることが一般的なパターンです。
（`run:` はシェルスクリプトのラッパー）

タスクとラッパースクリプトに関して言えば、次のようなルールで実行するパイプラインを構築します。

-	タスクファイルとラッパーシェルスクリプトに同じベースネームを与える

`01_task_hello_world` フォルダの中にある２つのファイルをみてください。

-	`task_show_uname.yml`
-	`task_show_uname.sh`

`fly` によってタスクファイルが直接的に実行されるとき、タスクへの入力として現在のフォルダが使用されます。これはラッパーシェルスクリプトが実行に利用できることを意味しています。

```
$ fly -t tutorial execute -c task_show_uname.yml
Connecting to 192.168.100.4:8080 (192.168.100.4:8080)
-                    100% |*******************************| 10240   0:00:00 ETA
initializing with docker:///busybox
running ./task_show_uname.sh
Linux mjgia714eg3 3.13.0-49-generic #83-Ubuntu SMP Fri Apr 10 20:11:33 UTC 2015 x86_64 GNU/Linux
succeeded
```
上で示した`running ./task_show_uname.sh`という出力は、`task_show_uname.yml`がタスク実行用のラッパースクリプトに処理を委譲したことを表しています。


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

タスクがラッパースクリプトを実行するためには、ラッパースクリプトへのアクセスする必要があります。
タスクがデータファイルを処理するためには、データファイルへアクセスする必要があります。

Concourseにおいて、タスクのそれらは`inputs`によって表現されます

`fly`によって直接的にタスクが実行されると同時に、`01_task_hello_world`の内側にある私たちのホストマシンから実行され、
現在のホストマシンのフォルダがConcourseにアップロードされ、`01_task_hello_world`と呼ばれる入力が利用可能になる。

入力があるジョブを後で見るとき、仕事とアウトプットはジョブの範囲内のタスクの中へインプットを通してリターンされる。

上に示した`inputs`のスニペットについて考慮すると、

```yaml
inputs:
- name: 01_task_hello_world
  path: .
```

これらは次のことを言っている。

1.	私は`01_task_hello_world` と呼ばれる入力フォルダを受け取りたい。
2.	私はそれを `.` フォルダに置きたい。(`.`は実行されるとき、タスクのルートフォルダとみなされる)

デフォルトでは、`path:`に何もなかった場合、入力自身と同じ名前のフォルダに置き換えられる。

`inputs`のリストが与えられたとき、（同じ名前のフォルダの中にある）`task_show_uname.sh`スクリプトは実行タスクのルートフォルダで利用可能になる。

これは次のような呼び出しを許可する

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

これらはコンコースのパイプライン（と何も変更がないこと）を表示し確認を求める。

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

変更確認
`fly set-pipeline` (もしくは `fly sp`)を実行するたびになにも変更がないことの確認を承諾するプロンプトが表示される。

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

前にある`pipeline.yml`にあるコンフィグレーションによってジョブのタスクを簡単に繰り返すことができる。結局、あなたが既に引っぱり出したリソースたちの中に含まれるジョブタスクを同じ場所に配置したいかもしれない。

これは少し複雑な"hello world" タスクの例であるが、私たちが実行したいタスクが事前に示した"01 - Hello World task"からのものであると仮定しよう。
それはgitリポジトリに保存されている。

`pipeline.yml`の中へ、チュートリアルのgitリポジトリにリソースとして加える。

```yaml
resources:
- name: resource-tutorial
  type: git
  source:
    uri: https://github.com/starkandwayne/concourse-tutorial.git
```

今、私たちはジョブをリソースとして消費することができる。　アップデートしましょう。

```yaml
jobs:
- name: job-hello-world
  public: true
  plan:
  - get: resource-tutorial
  - task: hello-world
    file: resource-tutorial/01_task_hello_world/task_hello_world.yml
```


最初の`get`は`resource-tutorial`というリソースを得るために必要だと、私たちの`plan:`で明示されています。
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

一つ目のステップは、gitリポジトリからトレーニング教材とチュートリアルを取得することです。この"resource"は`resource-tutorial`と呼ばれています。

このリソースはどんなジョブのビルドプランを含むタスクでも入力とすることができます。

二つ目のステップは、ユーザ定義のタスクを実行することです。UIの出力に表示された、タスク名`hello-world`が与えられます。タスク自身は
パイプラインの中で記述されません。その代わり、`resource-tutorial`の入力によって`01_task_hello_world/task_hello_world.yml`の中に記述されます。

パイプラインの外側に出して、YAMLファイルの中にタスクを記述することは利点と欠点がそれぞれあります。

利点は、作用するインプットリソースとマッチするようにタスクの振る舞いを修正できることが挙げられます。例えば、もし入力リソースがテスト付きのコードリポジトリであるならば、コードリポジトリはどのようにテストを持つ必要があるかの点でタスクファイルの同期を保ち続けることができます。

欠点は、`pipeline.yml`がどんなコマンドを実行するかを正確に説明しないことです。ファイルから理解できることが減るということです。
`pipeline.yml`ファイルが長くなればなるほど、全てのYAMLを読んで理解することは困難になるかもしれません

これらの選択をする場合、他のチームメンバの理解を考える必要があります。「このパイプラインは実際何をするんだ！」
ひとつのアイデアはタスクファイルの名前付けをどうするか考慮することです。例えば、ラッパースクリプトの実行内容によって名前付けを行うのも良いかもしれません。

その目的と振る舞いを説明する（長い）名前を使うことを考えましょう。

大切なことは`pipeline.yml`を読めるように作ることです。それはチーム/会社/プロジェクトの中で重要なオーケストレーションとになるでしょう。誰もがどのようにそれが実際動作するのかを知っているベキです。

### 04 - Get job output in terminal

The `job-hello-world` had terminal output from its resource fetch of a git repo and of the `hello-world` task running.

You can also view this output from the terminal with `fly`:

```
fly -t tutorial watch -j 03_resource_job/job-hello-world
```

The output will be similar to:

```
Cloning into '/tmp/build/get'...
e8c6632 Added trigger: true to autostart both jobs after update.
initializing with docker:///busybox
running echo hello world
hello world
succeeded
```

### 05 - Trigger a Job via the Concourse API

Our concourse in vagrant has an API running at `http://192.168.100.4:8080`. The `fly` CLI targets this endpoint by default.

We can trigger a job to be run using that API. For example, using `curl`:

```
curl http://192.168.100.4:8080/pipelines/03_resource_job/jobs/job-hello-world/builds -X POST
```

You can then watch the output in your terminal using `fly watch` from above:

```
fly -t tutorial watch -j 03_resource_job/job-hello-world
```

### 06 - Triggering jobs - the `time` resource

"resources are checked every minute, but there's a shorter (10sec) interval for determining when a build should run; time resource is to just ensure a build runs on some rough periodicity; we use it to e.g. continuously run integration/acceptance tests to weed out flakiness" - alex

The net result is that a timer of `2m` will trigger every 2 to 3 minutes.

### 20 - Available concourse resources

https://github.com/concourse?query=resource

-	[bosh-deployment-resource](https://github.com/concourse/bosh-deployment-resource) - deploy bosh releases as part of your pipeline
-	[semver-resource](https://github.com/concourse/semver-resource) - automated semantic version bumping
-	[bosh-io-release-resource](https://github.com/concourse/bosh-io-release-resource) - Tracks the versions of a release on bosh.io
-	[s3-resource](https://github.com/concourse/s3-resource) - Concourse resource for interacting with AWS S3
-	[git-resource](https://github.com/concourse/git-resource) - Tracks the commits in a git repository.
-	[bosh-io-stemcell-resource](https://github.com/concourse/bosh-io-stemcell-resource) - Tracks the versions of a stemcell on bosh.io.
-	[vagrant-cloud-resource](https://github.com/concourse/vagrant-cloud-resource) - manages boxes in vagrant cloud, by provider
-	[docker-image-resource](https://github.com/concourse/docker-image-resource) - a resource for docker images
-	[archive-resource](https://github.com/concourse/archive-resource) - downloads and extracts an archive (currently tgz) from a uri
-	[github-release-resource](https://github.com/concourse/github-release-resource) - a resource for github releases
-	[tracker-resource](https://github.com/concourse/tracker-resource) - pivotal tracker output resource
-	[time-resource](https://github.com/concourse/time-resource) - a resource for triggering on an interval
-	[cf-resource](https://github.com/concourse/cf-resource) - Concourse resource for interacting with Cloud Foundry

To find out which resources are available on your target Concourse you can ask the API endpoint `/api/v1/workers`:

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
