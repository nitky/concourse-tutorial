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

It will display the concourse pipeline (or any changes) and request confirmation:

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

You will be prompted to apply any configuration changes each time you run `fly set-pipeline` (or its alias `fly sp`)

```
apply configuration? (y/n):
```

Press `y`.

You should see:

```
pipeline created!
you can view your pipeline here: http://192.168.100.4:8080/pipelines/02helloworld
```

Go back to your browser and start the job manually. Click on `job-hello-world` and then click on the large `+` in the top right corner. Your job will run.

![job](http://cl.ly/image/3i2e0k0v3O2l/02-job-hello-world.gif)

Clicking the top-left "Home" icon will show the status of our pipeline.

### 03 - Tasks extracted into resources

It is easy to iterate on a job's tasks by configuring them in the `pipeline.yml` as above. Eventually you might want to colocate a job task with one of the resources you are already pulling in.

This is a little convoluted example for our "hello world" task, but let's assume the task we want to run is the one from "01 - Hello World task" above. It's stored in a git repo.

In our `pipeline.yml` we add the tutorial's git repo as a resource:

```yaml
resources:
- name: resource-tutorial
  type: git
  source:
    uri: https://github.com/starkandwayne/concourse-tutorial.git
```

Now we can consume that resource in our job. Update it to:

```yaml
jobs:
- name: job-hello-world
  public: true
  plan:
  - get: resource-tutorial
  - task: hello-world
    file: resource-tutorial/01_task_hello_world/task_hello_world.yml
```

Our `plan:` specifies that first we need to `get` the resource `resource-tutorial`.

Second we use the `01_task_hello_world/task_hello_world.yml` file from `resource-tutorial` as the task configuration.

Apply the updated pipeline using `fly set-pipeline -t tutorial -c pipeline.yml -p 03_resource_job`. #TODO find out how to do that better

Note: `fly` has shorter aliases for it's commands, `fly sp` is shorthand for `fly set-pipeline`

Or run the pre-created pipeline from the tutorial:

```
cd ../03_resource_job
fly sp -t tutorial -c pipeline.yml -p 03_resource_job
fly unpause-pipeline -t tutorial -p 03_resource_job
```

![resource-job](http://cl.ly/image/271z3T322l25/03-resource-job.gif)

After manually triggering the job via the UI, the output will look like:

![job-task-from-wrapper](http://cl.ly/image/0Q3m223v2l3M/job-task-from-wrapper.png)

The `job-hello-world` job now has two steps in its build plan.

The first step fetches the git repository for these training materials and tutorials. This is a "resource" called `resource-tutorial`.

This resource can now be an input to any task in the job build plan.

The second step runs a user-defined task. We give the task a name `hello-world` which will be displayed in the UI output. The task itself is not described in the pipeline. Instead it is described in `01_task_hello_world/task_hello_world.yml` from the `resource-tutorial` input.

There is a benefit and a downside to abstracting tasks into YAML files outside of the pipeline.

The benefit is that the behavior of the task can be modified to match the input resource that it is operating upon. For example, if the input resource was a code repository with tests then the task file could be kept in sync with how the code repo needs to have its tests executed.

The downside is that the `pipeline.yml` no longer explains exactly what commands will be invoked. Comprehension is potentially reduced. `pipeline.yml` files can get long and it can be hard to read and comprehend all the YAML.

Consider comprehension of other team members when making these choices. "What does this pipeline actually do?!"

One idea is to consider how you name your task files, and thus how you name the wrapper scripts that they invoke.

Consider using (long) names that describe their purpose/behavior.

Try to make the `pipeline.yml` readable. It will become important orchestration within your team/company/project; and everyone needs to know how it actually works.

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
