# ArgoCD セットアップ

## 環境

- Kubernetes: v1.35.1-gke.1396002 (GKE)
- ArgoCD Helm chart: 9.4.17

---

## 実行ログ

### 1. インストール

```bash
helmfile -e dev sync -l name=argocd
```

### 2. 動作確認

```bash
kubectl get pods -n argocd
```

### 3. UIへのアクセス

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

ブラウザで `http://localhost:8080` を開く。

- ユーザー: `admin`
- パスワード: 以下で取得

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```
