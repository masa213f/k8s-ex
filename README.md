# Kubernetesの動作確認

Kubernetesの動作確認用の手順を置いています。

## 動作確認環境

このリポジトリに保存されているマニフェストは、kind([https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/))で構築したKubernetesクラスタで動作確認しました。
環境が異なると動かない可能性あがります。特にホストOSが異なる場合は(例えばM1 Macとか)怪しい。

以下の環境で確認しています。

- ホストOS: Ubuntu 20.04 (x86_64)
- kind: v0.14.0
- Kubernetes
  - Client Version: v1.24.0
  - Server Version: v1.24.0

## 事前準備

1. `kind`コマンドをインストールする。
    - [https://kind.sigs.k8s.io/docs/user/quick-start#installation](https://kind.sigs.k8s.io/docs/user/quick-start#installation)
2. `kubectl`コマンドをインストールする。
    - [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
3. Kubernetesクラスタを立ち上げる。
    ```console
    $ kind create cluster
    $ kind export 
    ```

## 動作確認

1. [Pod](./pod)
   - コンテナのルートファイルシステムやPIDを確認してみる。
2. [ConfigMap/Secret](./configmap_secret)
   - ConfigMapとSecretを使ってみる。
3. [emptyDir](./emptydir)
   - `emptyDir`を使ってみる。
4. [RBAC](./rbac)
   - ServiceAccountに対してRBACを設定してみる。
