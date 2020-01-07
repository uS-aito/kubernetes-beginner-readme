# 概要
社内の勉強会で使用したDocker/Kubernetesハンズオン環境構築用のIaaSファイルです。  
勉強会は複数回開催されることが望ましいことと、参加者用の環境をいちいち手で作っていると問題が発生しそうだったので、IaC化しました。
現在作りかけ。

# 構成
## terraformフォルダ
Docker及びkubectl実行用のVMをデプロイします。usersというリスト変数に、ハンズオン参加者のユーザ名を列挙してください。人数分のVMをデプロイします。
## setup-shell-scriptフォルダ
terraform実行用に必要な環境変数の設定と、作成したVMにdockerやkubectl、AKSクラスタのcredential情報取得を行うスクリプトのフォルダです。

# 使い方
1. setup_env_var.shファイルを作成します。azureでサービスプリンシパルの発行とRBACの設定を行い、ID/secretを設定してください。
1. setup_env_var.shファイルを実行します。
    ```
    source ./setup_env_var.sh
    ```
1. terraformの変数を設定します。tfvarsファイルに必要な変数を設定してください。必須変数はユーザ名のリストのusersです。その他設定可能な変数はvariables.tfファイルを参照してください。
1. リソースをデプロイします。
    ```
    terraform init
    terraform plan
    terraform apply
    ```
1. 終わったらリソースを削除してください。
    ```
    terraform destroy
    ```

# Hands on
## 1. 環境の確認
まず、今日の勉強会で使用する環境が正常に動作していることを確認します。
1. 各自のVMにSSHでログインできることを確認してください。
1. dockerの動作を確認してください。
    ```
    $ docker version
    Client:
    Version:           18.09.0
    API version:       1.39
    Go version:        go1.10.4
    Git commit:        4d60db4
    Built:             Wed Nov  7 00:48:22 2018
    OS/Arch:           linux/amd64
    Experimental:      false

    Server: Docker Engine - Community
    Engine:
    Version:          18.09.0
    API version:      1.39 (minimum version 1.12)
    Go version:       go1.10.4
    Git commit:       4d60db4
    Built:            Wed Nov  7 00:19:08 2018
    OS/Arch:          linux/amd64
    Experimental:     true
    $
    ```
    Client、Serverの両部分が表示されることを確認してください。
1. kubectlコマンドの動作を確認してください。
    ```
    $ kubectl version
    Client Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.1", GitCommit:"b7394102d6ef778017f2ca4046abbaa23b88c290", GitTreeState:"clean", BuildDate:"2019-04-08T17:11:31Z", GoVersion:"go1.12.1", Compiler:"gc", Platform:"linux/amd64"}
    Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.1", GitCommit:"b7394102d6ef778017f2ca4046abbaa23b88c290", GitTreeState:"clean", BuildDate:"2019-04-08T17:02:58Z", GoVersion:"go1.12.1", Compiler:"gc", Platform:"linux/arm"}
    $
    ```
    Client、Serverの両部分が表示されることを確認してください。
## 2. コンテナの体験
この章では、従来のVMに対するコンテナの利便性を体験します。
1. まずはdockerコマンドとコンテナの実行を行います。以下のコマンドを実行してください。
    ```
    $ docker run hello-world

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:
    1. The Docker client contacted the Docker daemon.
    2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
        (amd64)
    3. The Docker daemon created a new container from that image which runs the
        executable that produces the output you are currently reading.
    4. The Docker daemon streamed that output to the Docker client, which sent it
        to your terminal.

    To try something more ambitious, you can run an Ubuntu container with:
    $ docker run -it ubuntu bash

    Share images, automate workflows, and more with a free Docker ID:
    https://cloud.docker.com/

    For more examples and ideas, visit:
    https://docs.docker.com/engine/userguide/

    $
    ```
    上記のような結果が得られれば、コマンドは正常に実行されました。  
    今の一連のコマンドで、hello-worldという名前のコンテナイメージを実行し、そのコンテナが一連の出力を表示し、仕事が終わったので停止しました。
1. 次はもう少し実用的、かつオプションの多い例を実行します。以下のコマンドを実行してください。
    ```
    $ docker run -d -p 8080:80 httpd
    e94a03462e96e641ae689be41e47329783d49d32a8c75ca1ac1820572f4714c5
    $ 
    ```
    次に、localhost:8080にHTTPリクエストを送信してください。
    ```
    $ curl localhost:8080
    <html><body><h1>It works!</h1></body></html>
    $
    ```
    Apacheの初期画面が確認できたかと思います。このコマンドではApacheを実行するコンテナを作成しました。過去、ESXiやIaaS上にApacheを実行するVMを建てたことのある人は多いと思いますが、それらと比較して圧倒的に早くApacheを実行する環境を作成できたことがわかるかと思います。  
    単純に比較すると、dockerがESXi、コンテナがVMに相当します。VMの作成からOSのインストール、Apacheのインストールが必要なVMに比べ、コンテナはapacheのバイナリとその周辺ライブラリくらいしか含んでいないため、コンテナイメージの取得も実行も高速です。  
    また、Apacheコンテナのイメージはインターネット上に公開されているので、インターネットに疎通さえしていればどのような端末からでも取得でき、dockerがインストールされていれば、そのイメージを実行できます。
## 3. kubernetesの体験
この章では、kubernetesの基本的な操作を体験するとともに、コンテナ(およびコンテナを実行するプラットフォームであるDocker)にはない機能を体験します。
1. kubectlコマンドはkubernetesクラスタを操作するためのコマンドです。まずは、現在kubernetesクラスタ上で実行されているコンテナの有無を確認します。以下のコマンドを実行してください。
    ```
    $ kubectl get pods
    No resources found in default namespace.
    $
    ```
    `kubectl get pods`コマンドは現在クラスタ上で実行されているコンテナの一覧を取得するコマンドです(コンテナとpodの違いはあとで説明されます)。現在、クラスタ上にはなんらコンテナが実行されていないため、その旨表示されました。
1. 次に、kubernetesクラスタ上にコンテナをデプロイします。クラスタ上に何らかのリソースをデプロイするには、manifestファイルを使用します。manifestファイルは展開するリソースの種類と望ましい状態を記述したyamlファイルです。まずはmanifestファイルを取得してください。
    ```
    $curl https://raw.githubusercontent.com/uS-aito/k8s-labo-manifests/master/deployment-3pods.yaml > deployment-3pods.yaml
    ```
    次に、取得したmanifestファイルを適用します。manifestファイルの適用は`kubectl apply -f <file>`で行います。
    ```
    $ kubectl apply -f ./deployment-3pods.yaml
    deployment.apps/httpd-3-deployment created
    $
    ```
    デプロイされたことを確認します。
    ```
    $ kubectl get pods
    NAME                                  READY   STATUS    RESTARTS   AGE
    httpd-3-deployment-5b48d6bbc8-6gf7x   1/1     Running   0          59s
    httpd-3-deployment-5b48d6bbc8-bmpxd   1/1     Running   0          59s
    httpd-3-deployment-5b48d6bbc8-d25qz   1/1     Running   0          59s
    $
    ```
1. 先ほどのyamlファイルの中身を閲覧します。文法はわからないにせよ、`replicas: 3`という文字列や、`containers: image: httpd`という文字列から、httpdのコンテナを3つ作れと言っているmanifestファイルであると類推できます。また、先ほどの`kubectl get pods`コマンドの実行結果から、manifestファイルの指示通りにhttpdのコンテナが3つ作られたことがわかります。
1. デプロイされたコンテナを一つ、強制的に停止させます。以下のコマンドを実行してください。
    ```
    $ kubectl delete pod <pod名>
    pod "<pod名>" deleted
    $
    ```
    <pod名>の部分は`kubectl get pods`コマンドで出力されたコンテナ名から一つ選んでください。次に、現在幾つのコンテナが実行されているか確認してみます。
    ```
    $ kubectl get pods
    NAME                                  READY   STATUS    RESTARTS   AGE
    httpd-3-deployment-5b48d6bbc8-bmpxd   1/1     Running   0          15m
    httpd-3-deployment-5b48d6bbc8-d25qz   1/1     Running   0          15m
    httpd-3-deployment-5b48d6bbc8-stx4r   1/1     Running   0          80s
    $
    ```
    3つコンテナが表示されたかと思います。説明にあった通り、kubernetesにはセルフヒーリングの機能が備わっており、3つあるべきコンテナの一つが何らかの理由で停止した場合、新しくコンテナを自動で起動します。
