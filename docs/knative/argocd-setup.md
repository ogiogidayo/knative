# ArgoCD セットアップ

## 環境

- Kubernetes: v1.35.1-gke.1396002 (GKE)
- ArgoCD Helm chart: 9.4.17

---

## 実行ログ

### 1. インストール

```bash
cd k8s
helmfile -e dev sync -l name=argocd
```

✅ 完了

### 2. 動作確認

```bash
$ kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          ...
argocd-applicationset-controller-...               1/1     Running   0          ...
argocd-dex-server-...                              1/1     Running   0          ...
argocd-notifications-controller-...               1/1     Running   0          ...
argocd-redis-...                                   1/1     Running   0          ...
argocd-repo-server-...                             1/1     Running   0          ...
argocd-server-...                                  1/1     Running   0          ...
```

✅ 全Pod Running

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

---

### 4. nginx-sample を ArgoCD で管理

#### Application マニフェスト (`k8s/argocd/apps/nginx-sample.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-sample
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ogiogidayo/knative
    targetRevision: HEAD
    path: k8s/kustomize/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### 適用

```bash
kubectl apply -f k8s/argocd/apps/nginx-sample.yaml
```

#### 動作確認

```bash
$ kubectl get application -n argocd
NAME           SYNC STATUS   HEALTH STATUS
nginx-sample   Synced        Healthy

$ kubectl get ksvc -n default
NAME           URL                                                   LATESTCREATED        LATESTREADY          READY   REASON
nginx-sample   http://nginx-sample.default.136.110.97.162.sslip.io   nginx-sample-00001   nginx-sample-00001   True
```

✅ ArgoCD から nginx-sample が Synced / Healthy

---

### 5. ArgoCD UI で確認

#### アプリ概要（リソースツリー全体）

![ArgoCD アプリ概要](./knative-argo/スクリーンショット%202026-03-29%2010.47.23.png)

- **APP HEALTH: Healthy** — アプリが正常稼働中
- **SYNC STATUS: Synced** — GitHub の HEAD と一致している
- **LAST SYNC: Sync OK** — 直近の同期が成功

左側のリソースツリーでは Knative が `ksvc` 1つから自動生成するリソース群が確認できる：

| リソース | 種別 | 説明 |
|---|---|---|
| `nginx-sample` | Application | ArgoCD が管理するアプリ |
| `nginx-sample` | Knative Service (ksvc) | Knative のトップレベルリソース |
| `nginx-sample` | Service (ExternalName) | Kourier へのエイリアス（外部トラフィックの入口） |
| `nginx-sample-00001` | Revision | デプロイごとに作られる不変スナップショット |
| `nginx-sample-00001` | Service (ClusterIP) | Activator 経由のルート |

#### リソースツリー詳細（Revision 配下）

![ArgoCD リソースツリー詳細](./knative-argo/スクリーンショット%202026-03-29%2010.47.49.png)

Revision `nginx-sample-00001` 配下に Knative が自動生成するリソース：

| リソース | 説明 |
|---|---|
| `nginx-sample-00001` (Deployment) | 実際の Pod を管理する Deployment |
| `nginx-sample-00001` (ReplicaSet) | Deployment が生成する ReplicaSet |
| `nginx-sample-00001` (Service) | Activator 経由のパブリックルート（port 80/443） |
| `nginx-sample-00001-private` (Service) | Pod への直接ルート（Activator バイパス用） |
| `nginx-sample-00001-cache-...` | Knative が内部管理に使う ConfigMap |

これらはすべて `ksvc` 1つを apply するだけで Knative が自動生成する。ArgoCD はそれらをすべて Git 管理対象として可視化している。

---

## 補足: selfHeal の挙動

`selfHeal: true` を設定しているため、クラスター上のリソースを手動で削除しても ArgoCD が Git の状態に戻す。
アプリを完全に削除したい場合は **ArgoCD Application を先に削除**してから ksvc を削除する。

```bash
# 正しい削除手順
kubectl delete application <name> -n argocd
kubectl delete ksvc <name> -n default
```
