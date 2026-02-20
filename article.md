---
title: eBPF × Grafana Beyla × kindで体感するゼロコード計装
tags: eBPF Kubernetes Observability Grafana OpenTelemetry
---

## はじめに

アプリケーションのObservabilityを実現するには、これまでOpenTelemetry SDKをコードに組み込む「手動計装」が一般的でした。しかし、言語ごとにSDKを導入し、コードを修正し、依存関係を管理する作業は無視できないコストです。

eBPF（extended Berkeley Packet Filter）は、Linuxカーネル内でサンドボックス化されたプログラムを実行する技術です。もともとはネットワークパケットのフィルタリング用でしたが、現在ではセキュリティ、ネットワーキング、そして**Observability**の領域で急速に活用が広がっています。

この記事では、eBPFを活用したGrafana Beylaを使い、**アプリケーションコードを一切変更せずに**分散トレーシングを実現する方法を、kind（Kubernetes in Docker）上のサンプルアプリで実際に手を動かしながら解説します。

## Grafana Beylaとは

[Grafana Beyla](https://grafana.com/docs/beyla/latest/)は、eBPFをベースとしたアプリケーション自動計装ツールです。2025年5月のGrafanaCON 2025で、Grafana LabsはBeylaのコアを[OpenTelemetryプロジェクトに寄贈](https://grafana.com/blog/2025/05/07/opentelemetry-ebpf-instrumentation-beyla-donation/)したことを発表しました。寄贈先での新プロジェクト名は**OpenTelemetry eBPF Instrumentation（OBI）**で、2025年11月に最初のalphaリリースが公開されています。現在のGrafana Beylaは、このupstreamプロジェクトのGrafana Labsディストリビューションという位置づけです。

**特徴:**

- **ゼロコード計装**: アプリケーションにSDKを追加する必要がない
- **言語非依存**: Go, Java, Python, Node.js, Rust, C/C++など、コンパイル言語・インタプリタ言語を問わず対応
- **カーネルレベルの計装**: eBPFによりシステムコールやネットワークI/Oをカーネル空間で直接フック
- **OpenTelemetry互換**: OTLPでトレース・メトリクスをエクスポート
- **豊富なプロトコル対応**: HTTP/HTTPS, HTTP/2, gRPC, SQL, Redis, MongoDB, Kafka, GraphQL, Elasticsearch/OpenSearchなど

BeylaはKubernetes上ではDaemonSetとして各ノードにデプロイされ、指定したnamespaceやポートのトラフィックを自動的に検出・計装します。

## 構成概要

今回構築する環境の全体像です。

```
┌─────────────────────────────────────────────────────────────┐
│  kind クラスタ (ebpf-demo)                                    │
│                                                              │
│  ┌─── namespace: app ────────────────────────────────────┐   │
│  │                                                       │   │
│  │  Client ──→ Nginx (:80) ──→ Go API (:8080) ──→ PostgreSQL (:5432) │
│  │             NodePort:30080                            │   │
│  └───────────────────────────────────────────────────────┘   │
│                          ↑ eBPF フック                        │
│  ┌─── namespace: observability ──────────────────────────┐   │
│  │                                                       │   │
│  │  Beyla ──→ OTel Collector ──→ Tempo ──→ Grafana       │   │
│  │  (DaemonSet)                        NodePort:30030    │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

- **Nginx**: リバースプロキシとして`/api/`へのリクエストをGo APIに転送
- **Go API**: `net/http` + `database/sql`のみで構成したシンプルなTODO CRUD API
- **PostgreSQL**: TODOデータの永続化
- **Beyla**: eBPFで`app` namespaceのポート8080, 5432のトラフィックを自動計装
- **OTel Collector**: Beylaからのテレメトリデータを受信しバッチ処理
- **Tempo**: 分散トレーシングのバックエンド
- **Grafana**: トレースの可視化UI

ポイントは、Go APIにはOpenTelemetry SDKを一切組み込んでいないことです。`net/http`と`database/sql`だけの素朴なコードを、Beylaが外部からカーネルレベルで計装します。

## 環境構築

### 前提条件

以下がインストールされていることを確認してください。

- Docker
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- kubectl

### サンプルリポジトリの取得

```bash
git clone https://github.com/<your-username>/ebpf-beyla-kind.git
cd ebpf-beyla-kind
```

### サンプルアプリの概要

Go APIのコードを見てみましょう（`app/api/main.go`の抜粋）。

```go
func main() {
    // PostgreSQL への接続（リトライ付き）
    dsn := fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
        getEnv("DB_HOST", "postgres"),
        getEnv("DB_PORT", "5432"),
        getEnv("DB_USER", "postgres"),
        getEnv("DB_PASSWORD", "postgres"),
        getEnv("DB_NAME", "postgres"),
    )
    // ... 接続処理 ...

    mux := http.NewServeMux()
    mux.HandleFunc("/health", handleHealth)
    mux.HandleFunc("/api/todos", handleTodos)
    mux.HandleFunc("/api/todos/", handleTodoByID)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

見ての通り、OpenTelemetry関連のインポートやコードは一切ありません。純粋な`net/http`サーバーです。

### Dockerイメージのビルド

```bash
docker build -t sample-api:local ./app/api
docker build -t sample-nginx:local ./app/nginx
```

### kindクラスタの作成

kindの設定ファイル（`k8s/kind-config.yaml`）:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080    # Nginx → localhost:8080
        protocol: TCP
      - containerPort: 30030
        hostPort: 3000    # Grafana → localhost:3000
        protocol: TCP
```

`extraPortMappings`により、kindノードのNodePortをホストマシンに公開しています。

```bash
kind create cluster --name ebpf-demo --config k8s/kind-config.yaml
```

### イメージのロード

kindクラスタはローカルのDockerイメージレジストリに直接アクセスできないため、明示的にロードします。

```bash
kind load docker-image sample-api:local sample-nginx:local --name ebpf-demo
```

### アプリケーションのデプロイ

```bash
# Namespace 作成
kubectl apply -f k8s/namespace.yaml

# PostgreSQL
kubectl apply -f k8s/postgres/

# Go API（PostgreSQL 起動を initContainer で待機）
kubectl apply -f k8s/api/

# Nginx（リバースプロキシ）
kubectl apply -f k8s/nginx/
```

Podがすべて Runningになるまで待ちます。

```bash
kubectl get pods -n app -w
```

```
NAME                        READY   STATUS    RESTARTS   AGE
postgres-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
api-xxxxxxxxxx-xxxxx        1/1     Running   0          25s
nginx-xxxxxxxxxx-xxxxx      1/1     Running   0          20s
```

### Observabilityスタックのデプロイ

```bash
# Observability 用 Namespace
kubectl apply -f k8s/observability/namespace.yaml

# Beyla（eBPF 計装エージェント）
kubectl apply -f k8s/observability/beyla/

# OpenTelemetry Collector
kubectl apply -f k8s/observability/otel-collector/

# Tempo（トレースバックエンド）
kubectl apply -f k8s/observability/tempo/

# Grafana（可視化）
kubectl apply -f k8s/observability/grafana/
```

```bash
kubectl get pods -n observability -w
```

```
NAME                              READY   STATUS    RESTARTS   AGE
beyla-xxxxx                       1/1     Running   0          30s
otel-collector-xxxxxxxxxx-xxxxx   1/1     Running   0          25s
tempo-xxxxxxxxxx-xxxxx            1/1     Running   0          20s
grafana-xxxxxxxxxx-xxxxx          1/1     Running   0          15s
```

## 動作確認

### APIへリクエスト送信

```bash
# ヘルスチェック
curl http://localhost:8080/health
# {"status":"ok"}

# TODO 作成
curl -X POST http://localhost:8080/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "eBPFを学ぶ"}'
# {"id":1,"title":"eBPFを学ぶ","completed":false,"created_at":"2026-02-20T..."}

# TODO 一覧取得
curl http://localhost:8080/api/todos
# [{"id":1,"title":"eBPFを学ぶ","completed":false,"created_at":"..."}]

# 複数回リクエストを送ってトレースデータを蓄積
for i in $(seq 1 10); do
  curl -s http://localhost:8080/api/todos > /dev/null
done
```

### Grafanaでトレースを確認

ブラウザで**http://localhost:3000**にアクセスします（認証なしでAdminとして自動ログインします）。

1. 左メニューの**Explore**を開く
2. データソースで**Tempo**を選択
3. **Search**タブで**Service Name**を選択し、`api`や`nginx`のサービスを確認
4. 任意のトレースをクリックして詳細を確認

トレースには以下のような情報が自動的に含まれています:

- **HTTPメソッドとパス**: `GET /api/todos`
- **ステータスコード**: `200`
- **レイテンシ**: 各スパンの所要時間
- **サービス間の依存関係**: Nginx → Go API → PostgreSQLのフロー

これらすべてが、**アプリケーションコードに一行も計装コードを追加せずに**取得できています。

## Beylaの仕組み深掘り

### eBPFによるカーネルレベルのフック

BeylaはeBPFプログラムをカーネルに注入し、以下のポイントにフックします:

1. **ソケット関連のシステムコール**: `accept`, `connect`, `read`, `write`, `close`など
2. **HTTP/gRPCパーサー**: カーネル空間でプロトコルを解析し、リクエスト/レスポンスのメタデータを抽出
3. **言語固有のフック**: Goの`net/http`やJavaの`java.net`など、ランタイムの既知関数にuprobesでアタッチ

```
┌──────────────────────────────────────────────┐
│  ユーザー空間                                   │
│                                               │
│  Go API        Nginx        PostgreSQL        │
│    │              │              │             │
├────┼──────────────┼──────────────┼─────────────┤
│  カーネル空間                                    │
│    │              │              │             │
│  eBPF プログラム（Beyla が注入）                   │
│    ↓              ↓              ↓             │
│  syscall フック: accept, read, write, close     │
│    │                                          │
│    ↓                                          │
│  eBPF Map（リングバッファ）                       │
│    │                                          │
├────┼──────────────────────────────────────────┤
│  ユーザー空間                                   │
│    ↓                                          │
│  Beyla プロセス → OTLP → OTel Collector        │
└──────────────────────────────────────────────┘
```

eBPFプログラムはカーネル内の検証器（verifier）を通過する必要があり、安全性が保証されています。無限ループやカーネルクラッシュを引き起こすコードはロードされません。

### Discovery設定の解説

BeylaのConfigMapを詳しく見てみましょう。

```yaml
discovery:
  services:
    - k8s_namespace: app
      open_ports: 8080,5432
```

この設定でBeylaは以下の動作をします:

1. Kubernetes APIを通じて`app` namespaceのPodを監視
2. ポート8080（Go API）と5432（PostgreSQL）をリッスンしているプロセスを検出
3. 検出したプロセスに対してeBPFプログラムをアタッチ

新しいPodがスケールアウトで追加された場合も、自動的に検出・計装されます。

### privileged + hostPIDが必要な理由

BeylaのDaemonSetでは以下の設定が必須です:

```yaml
spec:
  hostPID: true
  containers:
    - name: beyla
      securityContext:
        privileged: true
```

- **`hostPID: true`**: ホスト（ノード）のPID namespaceを共有し、他のPodのプロセスを認識するために必要
- **`privileged: true`**: eBPFプログラムのカーネルへのロードには`CAP_BPF`、`CAP_PERFMON`などの権限が必要。`privileged`はこれらを包含する

> **セキュリティについての補足**: 本番環境では`privileged: true`の代わりに、必要最小限のcapabilities（`CAP_BPF`, `CAP_PERFMON`, `CAP_NET_ADMIN`, `CAP_SYS_PTRACE`）を個別に指定することが推奨されます。

## 従来のSDK計装との比較

| 観点 | Beyla（eBPF） | OpenTelemetry SDK |
|---|---|---|
| **コード変更** | 不要 | 必要（SDKの初期化、スパン作成など） |
| **言語サポート** | 言語非依存（カーネルレベル） | 言語ごとにSDKが必要 |
| **導入コスト** | DaemonSetをデプロイするだけ | 各サービスにSDKを統合 |
| **カスタムスパン** | 不可（自動検出のみ） | 自由に追加可能 |
| **ビジネスロジックの計装** | 不可 | 可能（カスタム属性、イベント） |
| **トレースの粒度** | HTTP/gRPC/DBレベル | 関数・メソッドレベルまで可能 |
| **パフォーマンス影響** | 極めて小さい（カーネル空間で動作） | SDKの初期化・スパン処理のオーバーヘッド |
| **レガシーアプリ対応** | 得意（コード変更不可な環境） | 困難（コード変更が必須） |
| **セキュリティ要件** | privileged権限が必要 | 特別な権限不要 |

### 使い分けの指針

- **Beylaが向いているケース**:
  - レガシーアプリや修正不可なサードパーティサービスの計装
  - まずは最小限のコストでObservabilityを導入したい場合
  - マイクロサービス群を一括で計装したい場合

- **SDKが向いているケース**:
  - ビジネスロジック固有のスパンやメトリクスが必要な場合
  - カスタム属性やイベントで詳細なコンテキストを付与したい場合
  - セキュリティポリシーでprivilegedコンテナが許可されない環境

- **ハイブリッドアプローチ**: 実際のプロダクション環境では、Beylaでベースラインの計装を行い、重要なサービスにはSDKで詳細な計装を追加する組み合わせが効果的です。

## Beylaの最新動向: OpenTelemetryへの寄贈

2025年5月、Grafana LabsはBeylaのコアコードをOpenTelemetryプロジェクトに寄贈しました。これにより、eBPFベースのゼロコード計装はベンダー中立なOpenTelemetryエコシステムの一部として発展していくことになります。

### 経緯

| 時期 | 出来事 |
|---|---|
| 2024年10月 | Grafana LabsがOpenTelemetryコミュニティに[寄贈を提案](https://github.com/open-telemetry/community/issues/2406) |
| 2025年5月 | GrafanaCON 2025で正式発表、初回コードドロップ完了 |
| 2025年11月 | [OpenTelemetry eBPF Instrumentation（OBI）の最初のalphaリリース](https://opentelemetry.io/blog/2025/obi-announcing-first-release/) |

### OpenTelemetry eBPF Instrumentation（OBI）とは

寄贈先でのプロジェクト名は**OpenTelemetry eBPF Instrumentation（OBI）**です。Grafana Labs, Splunk, Coralogix, Odigosなど複数の組織がSIG（Special Interest Group）に参加し、共同で開発を進めています。

OBIはライブラリレベルではなく**プロトコルレベル**で計装を行うため、プログラミング言語やフレームワークに依存せず、単一のツールですべてのアプリケーションを一貫して計装できるのが特徴です。

### Grafana Beylaとの関係

Grafana Beylaは今後も存続しますが、その位置づけは変わります。

- Beyla 2.5以降、upstreamのOBIコードを直接vendoringする構成に移行
- 将来的にはGrafana固有の機能のみを含む薄いラッパーとなる予定
- メンテナーはupstreamでの開発を優先し、コードの重複を避ける方針

つまり、eBPFベースの自動計装のコア技術はOpenTelemetryコミュニティ全体の共有資産となり、Beylaはそのエンタープライズ向けディストリビューションという関係になります。

## クリーンアップ

```bash
kind delete cluster --name ebpf-demo
```

## まとめ

この記事では、Grafana Beylaを使ってkindクラスタ上のアプリケーションをゼロコードで計装する方法を実践しました。

- eBPFを使うことで、アプリケーションコードに一切手を加えずに分散トレーシングを実現できる
- BeylaはDaemonSetとして各ノードにデプロイするだけで、指定したnamespaceのサービスを自動検出・計装する
- 取得されるトレースにはHTTPメソッド、パス、ステータスコード、レイテンシなどが含まれ、サービス間のフローも可視化される
- ただし、ビジネスロジック固有の計装が必要な場合は従来のSDKとの併用が現実的

eBPFベースの自動計装はObservabilityの「最初の一歩」として極めて有効です。BeylaのOpenTelemetryへの寄贈により、この技術はベンダー中立なエコシステムの中で今後さらに発展していくことが期待されます。まずはBeylaでシステム全体の可視性を確保し、必要に応じてSDKで深掘りする — そんなアプローチを検討してみてはいかがでしょうか。

## 参考リンク

- [Grafana Beyla公式ドキュメント](https://grafana.com/docs/beyla/latest/)
- [OpenTelemetry eBPF Instrumentation（OBI）公式ドキュメント](https://opentelemetry.io/docs/zero-code/obi/)
- [OpenTelemetry eBPF Instrumentation GitHubリポジトリ](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation)
- [Beylaの寄贈についてのGrafana Labs公式ブログ](https://grafana.com/blog/2025/05/07/opentelemetry-ebpf-instrumentation-beyla-donation/)
- [eBPF.io](https://ebpf.io/) - eBPFの公式情報サイト
- [OpenTelemetry公式](https://opentelemetry.io/)
- [kind公式](https://kind.sigs.k8s.io/)
- [Grafana Tempo公式](https://grafana.com/docs/tempo/latest/)
