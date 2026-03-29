# Knative Eventing セットアップ

## 環境

- Kubernetes: v1.35.1-gke.1396002 (GKE)
- Knative Eventing: v1.21.1

---

## 実行ログ

### 1. CRD インストール

```bash
kubectl apply -k ./kustomize/base/knative-eventing/crds/
kubectl wait --for=condition=Established --all crd --timeout=60s
```

### 2. Eventing Core インストール

```bash
kubectl apply -k ./kustomize/base/knative-eventing/core/
```

### 3. 動作確認

```bash
kubectl get pods -n knative-eventing
```

---

## サンプル: CronJobSource → Broker → Trigger → event-display

### 構成

```
CronJobSource（10秒ごとにイベント発生）
      │ CloudEvents
      ▼
Broker（default）
      │
      ▼
Trigger（全イベントを通過）
      │
      ▼
event-display（ログにイベント内容を出力）
```

### デプロイ

```bash
kubectl apply -k ./kustomize/base/eventing-sample/
```

### イベント確認

```bash
kubectl logs -l app=event-display -n default --tail=50 -f
```
