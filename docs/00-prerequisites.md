# 00 - Prerequisites

## 学習目標

- ハンズオンに必要なツールを **すべて SDKMAN! / 公式バイナリ経由で導入** する
- バージョンを **完全に固定** して、再現性のある環境を作る

## 前提

| 項目 | 要件 | 備考 |
|------|------|------|
| OS | Linux / WSL2 (Ubuntu 22.04+ 推奨) | macOS でも可 (パッケージ名のみ読み替え) |
| メモリ | 8GB 以上 | Spring Boot のビルドと kind ノードで消費 |
| ディスク | 10GB 以上の空き | Docker イメージ・kind ノードで消費 |
| Docker | インストール済み・起動中 | Docker Desktop の WSL Integration 有効化、もしくは docker engine |
| インターネット | 必要 (各種ダウンロード) | |

確認:
```bash
docker ps
# → コンテナ一覧 (空でも OK)、エラーが出なければ Docker は OK
```

## 導入するツール

| ツール | バージョン | 役割 |
|--------|-----------|------|
| SDKMAN! | 5.x | Java / Gradle のバージョン管理 |
| Java | **25 (Temurin)** | Spring Boot 4.x の動作要件 |
| Gradle | **9.4.0** | ビルドシステム |
| kind | **v0.24.0** | ローカル Kubernetes クラスタ |
| kubectl | **v1.31+** | K8s 操作 CLI (Kustomize v5.x 内蔵) |
| argocd CLI | **v3.x** | Argo CD への CLI 接続 |

---

## Step 0-① SDKMAN! インストール

```bash
# SDKMAN! インストール (任意のディレクトリで OK)
curl -s "https://get.sdkman.io" | bash

# シェル再読み込み
source "$HOME/.sdkman/bin/sdkman-init.sh"

# 確認
sdk version
```

**期待される出力**:
```
SDKMAN!
script: 5.x.x
native: x.x.x (linux x86_64)
```

[SDKMAN! 公式](https://sdkman.io/install)

---

## Step 0-② Java 25 (Temurin) インストール

```bash
# 利用可能な Java 25 一覧
sdk list java | grep -E "25.*tem"

# Temurin Java 25 (例: 25.0.1-tem) をインストール
sdk install java 25.0.1-tem
# → "Do you want java 25.0.1-tem to be set as default?" → Y

# 確認
java --version
javac --version
```

**期待される出力**:
```
$ java --version
openjdk 25.0.1 2025-xx-xx
OpenJDK Runtime Environment Temurin-25.0.1+xx (build 25.0.1+xx)
OpenJDK 64-Bit Server VM Temurin-25.0.1+xx (build 25.0.1+xx, mixed mode, sharing)
```

> **代替**: Amazon Corretto 25 の場合は `25.0.1-amzn` でも OK (本ハンズオンは Temurin / Corretto どちらでも動作確認済)。

---

## Step 0-③ Gradle 9.4.0 インストール

```bash
sdk install gradle 9.4.0
# → "Do you want gradle 9.4.0 to be set as default?" → Y

gradle --version
```

**期待される出力 (主要部分)**:
```
------------------------------------------------------------
Gradle 9.4.0
------------------------------------------------------------
Launcher JVM:  25.0.1 (Eclipse Adoptium 25.0.1+xx)
Daemon JVM:    /home/user/.sdkman/candidates/java/25.0.1-tem
```

> Gradle Wrapper (`./gradlew`) はプロジェクト固有のスクリプト。ここでインストールする system gradle は **Wrapper の生成にのみ使う** (実プロジェクト実行は wrapper 経由)。

---

## Step 0-④ kind v0.24.0 インストール

公式バイナリから直接ダウンロード:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind --version
```

**期待される出力**:
```
kind version 0.24.0
```

[kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)

---

## Step 0-⑤ kubectl インストール

公式バイナリから直接ダウンロード (latest stable):

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

kubectl version --client
```

**期待される出力例**:
```
Client Version: v1.31.x
Kustomize Version: v5.x.x
```

> `kubectl version` の `Kustomize Version` は **後の章で重要**。Kustomize は kubectl 内蔵で別ツールが不要。

[kubectl Installation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

---

## Step 0-⑥ argocd CLI インストール

```bash
ARGOCD_VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/download/v${ARGOCD_VERSION}/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

argocd version --client
```

**期待される出力例**:
```
argocd: v3.x.x+xxxxxxx
  BuildDate: ...
  GitCommit: ...
  GoVersion: go1.x.x
  Platform: linux/amd64
```

`--client` を付けないとサーバー接続を試みてエラーになる点に注意 (まだサーバーが立っていない)。

---

## 全ツール一括確認

```bash
echo "--- Docker ---" && docker --version && \
echo "--- SDKMAN! ---" && sdk version && \
echo "--- Java ---" && java --version && \
echo "--- Gradle ---" && gradle --version | head -3 && \
echo "--- kind ---" && kind --version && \
echo "--- kubectl ---" && kubectl version --client && \
echo "--- argocd ---" && argocd version --client | head -2
```

すべてエラーなく出れば次の章へ進めます。

---

## 落とし穴 (トラブルシューティング)

| 症状 | 原因 | 対処 |
|------|------|------|
| `sdk: command not found` | シェル再読み込みされていない | `source "$HOME/.sdkman/bin/sdkman-init.sh"` を実行、もしくは新規ターミナル開く |
| `Cannot connect to the Docker daemon` | Docker 未起動 / WSL Integration 未設定 | Docker Desktop を起動、Settings → Resources → WSL Integration で対象 distro を ON |
| `apt install gradle` で 4.x が入る | apt のパッケージは古い | snap や SDKMAN! を使う (本ハンズオンは SDKMAN! 推奨) |
| `kubectl` の Kustomize バージョンが 4.x 以下 | kubectl が古い | v1.27+ (Kustomize v5+) にアップデート |

---

## 一次情報

- [SDKMAN! Installation](https://sdkman.io/install) - SDKMAN! 公式
- [Eclipse Temurin (Docker Hub)](https://hub.docker.com/_/eclipse-temurin) - Java 配布
- [Gradle User Guide](https://docs.gradle.org/current/userguide/userguide.html) - Gradle 公式
- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) - kind 公式
- [Install kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) - kubectl 公式
- [Argo CD Installation](https://argo-cd.readthedocs.io/en/stable/cli_installation/) - argocd CLI 公式

---

[← 戻る (README)](../README.md) | [次へ → 01 Environment Setup](01-environment-setup.md)
