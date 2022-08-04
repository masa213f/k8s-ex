# Podの動作確認

2つのコンテナで構成されるPodを使い、コンテナのルートファイルシステムやPIDを確認します。

## 手順概略

1. 2つのコンテナで構成されるPodを作る
2. コンテナ内のプロセスやファイルを確認する
3. 一方のコンテナ内にファイルを作り、もう一方のコンテナでファイルを探してみる
4. ファイルを作ったコンテナを終了(再起動)させる

## 詳細手順

### 準備

動作確認で使うNamespaceを作ります。

```console
$ kubectl create namespace ex-pod
namespace/ex-pod created
```

### 1. 2つのコンテナで構成されるPodを作る

次のマニフェストを適用します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  # Ubuntu 22.04 LTS (Jammy Jellyfish)
  - name: my-jammy
    image: ubuntu:jammy
    command: ["bash", "-c", "trap 'kill $(jobs -p)' EXIT; sleep 9999"]

  # Ubuntu 20.04.4 LTS (Focal Fossa)
  - name: my-focal
    image: ubuntu:focal
    command: ["bash", "-c", "trap 'kill $(jobs -p)' EXIT; sleep 8888"]

  # Restart each container when exits.
  restartPolicy: Always
```

```console
$ kubectl apply -n ex-pod -f <マニフェストのパス>
pod/test-pod created
```

> **Note**
> 
> `sleep`コマンドがPID 1になってしまうと、コンテナ再起動の確認ができないため、少し変わったコマンドにしています。
> 単にUbuntuのコンテナを動かすだけなら、`sleep infinity`だけでもOKです。

Podのステータスが`Running`になることを確認しましょう。

```console
$ kubectl get pod -n ex-pod
NAME              READY   STATUS    RESTARTS   AGE
test-pod          2/2     Running   0          21s
```

### 2. コンテナ内のプロセスやファイルを確認する

1つめのコンテナ`my-jammy`に入ってファイルを確認してみましょう。

```console
# コンテナを指定しない場合、デフォルトコンテナ(特に指定がない場合は1つめのコンテナ)。
$ kubectl exec -it -n ex-pod test-pod -- bash
Defaulted container "my-jammy" out of: my-jammy, my-focal

# /etc/os-releaseを見ると、Ubuntu 22.04であることが分かります。
root@test-pod:/# grep VERSION= /etc/os-release
VERSION="22.04.1 LTS (Jammy Jellyfish)"

# psコマンドを実行すると、bashとsleep 9999が実行されていることが分かります。
root@test-pod:/# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4352  3212 ?        Ss   13:38   0:00 bash -c trap 'kill $(jobs -p)' EXIT; sleep 9999
root          13  0.0  0.0   2780  1056 ?        S    13:38   0:00 sleep 9999
root          14  0.0  0.0   4616  3808 pts/0    Ss   13:40   0:00 bash
root          23  0.0  0.0   7052  1572 pts/0    R+   13:41   0:00 ps aux
```

## 3. 一方のコンテナ内にファイルを作り、もう一方のコンテナでファイルを探してみる

手順2の続きで、`my-jammy`コンテナの中で、ファイルを作ってみましょう。

```console
# ファイルを作ってみる。
root@test-pod:/# echo Hello > /tmp/from-jammy
root@test-pod:/# cat /tmp/from-jammy
Hello
```

次に、2つめのコンテナ`my-focal`に入って、`my-jammy`内で作ったファイルを探してみましょう。

```console
# コンテナ名を指定(-c my-focal)して、my-focalコンテナに入る
$ kubectl exec -it -n ex-pod test-pod -c my-focal -- bash

# Ubuntuのバージョンを確認してみる
root@test-pod:/# grep VERSION= /etc/os-release
VERSION="20.04.4 LTS (Focal Fossa)"

# my-jammyと異なるコマンドが、同じPIDで実行されている。(PID 1以外はずれる場合がある。)
root@test-pod:/# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   3976  2960 ?        Ss   13:38   0:00 bash -c trap 'kill $(jobs -p)' EXIT; sleep 8888
root          13  0.0  0.0   2508   512 ?        S    13:38   0:00 sleep 8888
root          14  0.0  0.0   4108  3496 pts/0    Ss   13:47   0:00 bash
root          23  0.0  0.0   5892  3012 pts/0    R+   13:47   0:00 ps aux

# my-jammyで作ったファイルがない
root@test-pod:/# ls /tmp/from-jammy
ls: cannot access '/tmp/from-jammy': No such file or directory
```

### 4. ファイルを作ったコンテナを終了(再起動)させる

まず、新しくコンソールを2つ起動し、`kubectl -n ex-pod get pod -w`を実行しておきます。
`-w(--watch)`はリソース状態を見続けて、変化があると出力してくれるコマンドです。

```console
$ kubectl -n ex-pod get pod -w
NAME              READY   STATUS    RESTARTS   AGE
test-pod          2/2     Running   0          2m45s
(コマンドが起動したままになる)
```

次に、`my-jammy`コンテナに入り、PID 1のプロセスを終了させます。

```console
$ kubectl exec -it -n ex-pod test-pod -- bash
Defaulted container "my-jammy" out of: my-jammy, my-focal
root@test-pod:/# kill 1
command terminated with exit code 137
```

先ほど起動した、kubectlの状態を見ると、一度Errorになった後、再度Runningになっていることがわかる。
```console
$ kubectl get pod -n ex-pod -w
NAME       READY   STATUS    RESTARTS   AGE
test-pod   2/2     Running   0          2m45s
test-pod   1/2     Error     0          2m55s   # コンテナのエラーを検出
test-pod   2/2     Running   1 (2s ago)   2m56s # コンテナを再起動
```

再起動後の`my-jammy`コンテナで、ファイルを確認する。

```
$ kubectl exec -it -n ex-pod test-pod -- bash
Defaulted container "my-jammy" out of: my-jammy, my-focal
root@test-pod:/# ls /tmp/
(表示なし)
```

- コンテナのプロセスが終了したら、Podの設定`.spec.restartPolicy`に従い、コンテナが再起動される。
- コンテナが再起動されると、ルートファイルシステムは初期化される。

### 加えて

以下をやってみると面白いです。

1. `.spec.shareProcessNamespace`を`true`にして、コンテナ内のプロセス一覧を見る。
2. `.spec.restartPolicy`を`Never`にして、コンテナ内の PID 1 のプロセスを終了した場合。
