# Knative Serving セットアップ

## 環境

- Kubernetes: v1.35.1-gke.1396002 (GKE)
- Node: gke-knative-poc-clus-default-node-poo-487ff77a-sgs3
- Knative Serving: v1.21.2
- Knative net-kourier: v1.20.1

---

## 実行ログ

### クラスター接続確認

```bash
$ kubectl get nodes
NAME                                                  STATUS   ROLES    AGE   VERSION
gke-knative-poc-clus-default-node-poo-487ff77a-sgs3   Ready    <none>   12m   v1.35.1-gke.1396002
```

---

### 1. CRD インストール

```bash
kubectl apply -k k8s/kustomize/base/knative/crds/
kubectl wait --for=condition=Established --all crd --timeout=60s
```

✅ 完了

### 2. Serving Core + Kourier インストール

Kourierの設定（`ingress-class`）もパッチとして同時適用。

```bash
kubectl apply -k k8s/kustomize/base/knative/core/
```

✅ 完了

### 3. Magic DNS（sslip.io）設定

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.21.2/serving-default-domain.yaml
```

✅ 完了

### 動作確認

```bash
$ kubectl get pods -n knative-serving
NAME                                      READY   STATUS    RESTARTS   AGE
activator-74fb7d4d7d-7z88h                1/1     Running   0          110s
autoscaler-5dd9bc5ddf-47blz               1/1     Running   0          110s
controller-6cb4f8dfd-b72dl                1/1     Running   0          109s
default-domain-7xqtp                      1/1     Running   0          28s
net-kourier-controller-55985f4b65-bfrsw   1/1     Running   0          109s
webhook-5f9bf9cbc4-58vbx                  1/1     Running   0          109s
```

✅ 全Pod Running

---

### 4. サンプルアプリデプロイ

```bash
$ kubectl apply -k ./kustomize/base/myapp/
Warning: Kubernetes default value is insecure, Knative may default this to secure in a future release: spec.template.spec.containers[0].securityContext.allowPrivilegeEscalation, spec.template.spec.containers[0].securityContext.capabilities, spec.template.spec.containers[0].securityContext.runAsNonRoot, spec.template.spec.containers[0].securityContext.seccompProfile
service.serving.knative.dev/myapp created
```

✅ 完了（Warningはセキュリティコンテキスト未設定の注意、動作には影響なし）
