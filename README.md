# Mac上に複数のDockerホストを用意し，Swarmクラスタを構築する
多くのリクエストをさばくには，複数のコンテナを複数のホストに配置させる必要があります．  
そこで，今回はお勉強としてまずMac上に，そのような環境を構築していきます．

## Docker Swarm
`Docker Swarm`は，複数の`Docker`ホストを束ねてクラスタ化するためのツールで，`kubernetes`のようなコンテナオーケストレーションシステムの一つです．コンテナオーケストレーションとは，複数のコンテナをに対する運用管理作業のことと認識しております．時代は，`kubernetes`ですね．  
さて，今回は，`Docker Swarm`を用いてクラスタ(`Swarm`クラスタ)を構築していきます．

## Macに，複数のDockerホストを用意する．
`Docker for Mac`では，インストールされる`Docker`は一つなので，色々して複数台用意します．  
複数台用意する方法は，クラウドサービスだったり，`Docker Machine`を用いることでできそうです．とても大変です．  
なので，今回は一番手っ取り早くできそうな`Docker in Docker`(略して`dind`)という仕組みを用います．`dind`は，`Docker`ホストとして機能する`Docker`コンテナを複数個立てることができます．

## 作成するコンテナ
* registry ×1
* manager ×1
* worker ×3

## docker-compose.ymlを書いて立ち上げる
[ここ](https://github.com/mytheta/DockerSwarm/blob/master/docker-compose.yml)を参照

```
docker-compose up -d
```

## コンテナが立ち上がっているかみてみる．
`docker comtainer ls`でみてみる．

```
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
70f38edc0918        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker03
87b56e609082        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker02
7f1d05c3fb9e        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker01
1951cabcf225        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 3375/tcp, 0.0.0.0:9000->9000/tcp, 0.0.0.0:8000->80/tcp   manager
052ac84438cb        registry:2.6             "/entrypoint.sh /etc…"   4 minutes ago       Up 4 minutes        0.0.0.0:5000->5000/tcp                                             registry
```
立ち上がっていますね．  
ただ，このままでは，協調してクラスタとして動作はしていません．  
クラスタを管理する役割を担うmanagerの設定を行います．  
ホストからmanagerコンテナに対して`docker swarn init`をして，`Swarn`のmanagerに設定します．  
``` 
$ docker container exec -it manager docker swarm init
Swarm initialized: current node (p03f5rr1mbcxgsubls54vwiec) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1eo2ygi371y22ixjkhdp6i32qiq47f8udwyol6u2d9e2204907-16vyeunwd6aq7v6kfqqm6ngzu 172.21.0.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
`token`が発行されていますね．これは，`Swarn`クラスタの`worker`として登録するのに必要です．  
早速，`Swarn`クラスタに`worker01`,`worker02`,`worker03`を登録していきます．  
以下のように`worker02`,`worker03`にもコマンドを叩いてください．  
コンテナ同士は，`compose`で作成されているため，名前解決できますので，`manager:2377`に対して`join`トークンを使います．  

```
$ docker container exec -it worker01 docker swarm join \
> --token SWMTKN-1-1eo2ygi371y22ixjkhdp6i32qiq47f8udwyol6u2d9e2204907-16vyeunwd6aq7v6kfqqm6ngzu manager:2377
This node joined a swarm as a worker.
```

続いて，Dockerレジストリにイメージをpushします．  
ここでは，`yutsuki/echo`のイメージを使います．  
その前にタグ付けを行います．  
```
$ docker image tag yutsuki/echo:latest localhost:5000/yutsuki/echo:latest
 ```
 
このようにタグ付けすることで，指定のレジストリにpushできます．  

registryコンテナにイメージをpush  
```
 $ docker image push localhost:5000/yutsuki/echo:latest
The push refers to repository [localhost:5000/yutsuki/echo]
e64be2cabced: Pushed
7ce979cdd8d6: Pushed
186d94bd2c62: Pushed
24a9d20e5bee: Pushed
e7dc337030ba: Pushed
920961b94eb3: Pushed
fa0c3f992cbd: Pushed
ce6466f43b11: Pushed
719d45669b35: Pushed
3b10514a95be: Pushed
latest: digest: sha256:624945c796c629f049bcae1809cf4ebddb52cec1b0a91dfa80603948383e4cb9 size: 2417
```
できていますね．  
では，workerコンテナがregistryコンテナからDockerイメージをpullできるか確認してみます．  

```
 $ docker container exec -it worker01 docker image pull registry:5000/yutsuki/echo:latest

latest: Pulling from yutsuki/echo
55cbf04beb70: Pull complete
1607093a898c: Pull complete
9a8ea045c926: Pull complete
d4eee24d4dac: Pull complete
9c35c9787a2f: Pull complete
8b376bbb244f: Pull complete
0d4eafcc732a: Pull complete
186b06a99029: Pull complete
4017f5c3a166: Pull complete
d6c72ebefb82: Pull complete
Digest: sha256:624945c796c629f049bcae1809cf4ebddb52cec1b0a91dfa80603948383e4cb9
Status: Downloaded newer image for registry:5000/yutsuki/echo:latest

$ docker container exec -it worker01 docker image ls

REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
registry:5000/yutsuki/echo   latest              7479f1250d86        39 minutes ago      750MB
```
無事にできていますね．  
次は，`Service`の設定をしていきます．  

## Service
Serviceとは，公式リファレンスによると  
> サービスは、 swarm 上でアプリケーション・コンテナをどのように実行するかの定義です。最も基本的なレベルのサービス定義とは、swarm 上でどのコンテナ・イメージを実行するか、そして、どのコマンドをコンテナで実行するかです。オーケストレーションの目的は「望ましい状態（desired state）」としてサービスを定義することです。つまり、いくつのコンテナをタスクとして実行するか、コンテナをデプロイする条件（constraint）を指します。

ん．．  
コンテナをどう制御するかという単位ぐらいに捉えております．  
より理解していくために，`Service`を実際に作っていきます．  
イメージには，`registry`コンテナに`push`してある`registry:5000/yutsuki/echo:latest`を使います．`Service`は`manager`コンテナ内から`docker service create`で行います．  

```
$ docker container exec -it manager \
> docker service create --replicas 1 --publish 8000:8080 --name echo registry:5000/yutsuki/echo:latest
k4raze392hep95aezt6ej8m23
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

実際に立ち上がってるか見てみます．
```
$ docker container exec -it manager docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
k4raze392hep        echo                replicated          1/1                 registry:5000/yutsuki/echo:latest   *:8000->8080/tcp
```
一つ作成されていますね．`Service`は，レプリカ(コンテナの複製？)の数を制御できます．複数のノードをまたいでコンテナの数を複製できるため，スケールアウトする際に役立ちます．6個レプリカを作ってみます．
```
$ docker container exec -it manager docker service scale echo=6
echo scaled to 6
overall progress: 6 out of 6 tasks
1/6: running   [==================================================>]
2/6: running   [==================================================>]
3/6: running   [==================================================>]
4/6: running   [==================================================>]
5/6: running   [==================================================>]
6/6: running   [==================================================>]
verify: Service converged
```
確認してみます．

```
$ docker container exec -it manager docker service ps echo | grep Running
finsmpqym219        echo.1              registry:5000/yutsuki/echo:latest   7f1d05c3fb9e        Running             Running 5 minutes ago
d6pga7qdjej2        echo.2              registry:5000/yutsuki/echo:latest   70f38edc0918        Running             Running about a minute ago
l1a84dhqvr05        echo.3              registry:5000/yutsuki/echo:latest   7f1d05c3fb9e        Running             Running 2 minutes ago
z8rtde8f9m58        echo.4              registry:5000/yutsuki/echo:latest   1951cabcf225        Running             Running about a minute ago
umejsy44yk4t        echo.5              registry:5000/yutsuki/echo:latest   87b56e609082        Running             Running about a minute ago
blh595dbewu5        echo.6              registry:5000/yutsuki/echo:latest   70f38edc0918        Running             Running about a minute ago
```

6個の`echo`コンテナが実行されていますね．  
`NODE`の項目をみてみると，`Service`によって`Swarm`クラスタのノードに分散して配置されていることがわかります．  
### 具体的にみてみます．
`echo1``echo3`の`7f1d05c3fb9e`のノードは，`worker01`,
`echo2``echo6`の`70f38edc0918`は，`worker03`,`echo4`の`1951cabcf225`は，`manager`, `echo5`の`87b56e609082 `は，`worker02`という対応となっています．  
`manager`も実行されるんですね．なんとなく，`Service`でレプリカ数を制御できることがわかりました．  
 
## Serviceの削除
```
$ docker container exec -it manager docker service rm echo
```

デプロイした`service`は，`docker service rm サービス名`で削除できます．

## Stack
次に，`Stack`をやっていきます．  
ここら辺で，書くのがつらくなってきました．．頑張ります．  
### Stackについて
`Stack`は複数の`Service`をグルーピングした単位であり，アプリケーションの全体の構成を定義します． `Stack`は，`Service`で解決できない問題を解決します．一つのアプリケーションイメージしか扱うことができなかった`Service`に対して，`Stack`は，複数の`Service`を扱うことができます．  
そしては，`Stack`によってデプロイされる`Service`群は`overlay`ネットワークに所属します．

## やること
`nginx`と`api`の二つの`Service`群の`Stack`をデプロイする．

## ch03-webapi.ymlを書く
Stackの構成を`ch03-webapi.yml`に書いていきます．
[ここ](https://github.com/mytheta/DockerSwarm/blob/master/stack/ch03-webapi.yml)を参照

### overlayネットワークの作成
まず，`nginx`と`api`の`Service`が同一の`overlay`ネットワークに所属させる必要があります．
~~`Stack`はなにも設定しなければ，`Stack`の数だけ`overlay`ネットワークが作成されてしまいます．~~

```
$ docker container exec -it manager docker network create --driver=overlay --attachable ch03
av5rc5ua81sdrxztjvlyre06e
```

では，デプロイします．
```
$ docker container exec -it manager docker stack deploy -c /stack/ch03-webapi.yml echo
Creating service echo_nginx
Creating service echo_api
```

`echo`スタックの`Service`一覧をみてみます．

```
docker container exec -it manager docker stack services echo
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
sasttmtcxinj        echo_nginx          replicated          3/3                 gihyodocker/nginx-proxy:latest
vftzjxzscfw2        echo_api            replicated          3/3                 registry:5000/yutsuki/echo:latest
```
無事サービスができてますね．
続いて，デプロイされたコンテナを確認します．

```
$ docker container exec -it manager docker stack ps echo
ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE               ERROR               PORTS
cl82joavweho        echo_api.1          registry:5000/yutsuki/echo:latest   7f1d05c3fb9e        Running             Running about an hour ago
nuaw9urce0mb        echo_nginx.1        gihyodocker/nginx-proxy:latest      87b56e609082        Running             Running about an hour ago
bxp8t35dc3b1        echo_api.2          registry:5000/yutsuki/echo:latest   87b56e609082        Running             Running about an hour ago
igs9684srq5n        echo_nginx.2        gihyodocker/nginx-proxy:latest      70f38edc0918        Running             Running about an hour ago
tamu1kt2ndnh        echo_api.3          registry:5000/yutsuki/echo:latest   70f38edc0918        Running             Running about an hour ago
6rcb1ez1c7mq        echo_nginx.3        gihyodocker/nginx-proxy:latest      7f1d05c3fb9e        Running             Running about an hour ago
```
これを，可視化してみます．
`visualizer.yml`を作成して，`Stack`としてデプロイします．

```
$ docker container exec -it manager docker stack deploy -c /stack/visualizer.yml visualizer
Creating network visualizer_default
Creating service visualizer_app
```
`http://localhost:9000/`でみてみましょう．
![visualizer](https://github.com/mytheta/DockerSwarm/blob/master/visualizer.png)

うまくできてますね．
最初につくった`Service`で作ったレプリカの`echo`が6個,
今作った`echo_nginx`と`echo_api`が三つずつありますね．
あと，`visualizer`が一つですね．
