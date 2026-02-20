# eBPF × Beyla × kind サンプル

Grafana Beyla（eBPF）によるゼロコード計装を kind クラスタ上で体験するサンプルリポジトリです。

## アーキテクチャ

```
Client → Nginx (リバースプロキシ) → Go API → PostgreSQL

Beyla (eBPF DaemonSet) → OTel Collector → Tempo → Grafana
```

## 前提条件

- Docker
- [kind](https://kind.sigs.k8s.io/)
- kubectl

## セットアップ

### 1. Docker イメージのビルド

```bash
docker build -t sample-api:local ./app/api
docker build -t sample-nginx:local ./app/nginx
```

### 2. kind クラスタの作成

```bash
kind create cluster --name ebpf-demo --config k8s/kind-config.yaml
```

### 3. イメージのロード

```bash
kind load docker-image sample-api:local sample-nginx:local --name ebpf-demo
```

### 4. アプリケーションのデプロイ

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres/
kubectl apply -f k8s/api/
kubectl apply -f k8s/nginx/
```

### 5. Observability スタックのデプロイ

```bash
kubectl apply -f k8s/observability/namespace.yaml
kubectl apply -f k8s/observability/beyla/
kubectl apply -f k8s/observability/otel-collector/
kubectl apply -f k8s/observability/tempo/
kubectl apply -f k8s/observability/grafana/
```

### 6. Pod の起動確認

```bash
kubectl get pods -n app
kubectl get pods -n observability
```

## 動作確認

### API へリクエスト

```bash
# ヘルスチェック
curl http://localhost:8080/health

# TODO 作成
curl -X POST http://localhost:8080/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "eBPFを学ぶ"}'

# TODO 一覧
curl http://localhost:8080/api/todos

# TODO 単体取得
curl http://localhost:8080/api/todos/1
```

### Grafana でトレース確認

ブラウザで http://localhost:3000 にアクセスし、左メニューの Explore から Tempo データソースを選択してトレースを検索できます。

## クリーンアップ

```bash
kind delete cluster --name ebpf-demo
```

## ディレクトリ構成

```
.
├── app/
│   ├── api/          # Go API サーバー（TODO CRUD）
│   └── nginx/        # Nginx リバースプロキシ
└── k8s/
    ├── kind-config.yaml
    ├── namespace.yaml
    ├── postgres/     # PostgreSQL
    ├── api/          # Go API
    ├── nginx/        # Nginx
    └── observability/
        ├── beyla/          # Grafana Beyla（eBPF 計装）
        ├── otel-collector/ # OpenTelemetry Collector
        ├── tempo/          # Grafana Tempo（トレースバックエンド）
        └── grafana/        # Grafana（可視化）
```
