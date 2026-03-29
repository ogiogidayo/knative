# Knative リソース構成図

## ネームスペース全体図

```
外部トラフィック
      │
      ▼ http://myapp.default.136.110.97.162.sslip.io
┌─────────────────────────────────────────────────────┐
│ namespace: kourier-system                           │
│                                                     │
│  svc/kourier (LoadBalancer: 136.110.97.162)         │
│       │ 80/443                                      │
│       ▼                                             │
│  pod/3scale-kourier-gateway  ←── HPA (1〜10)        │
└───────────────┬─────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────┐
│ namespace: knative-serving                          │
│                                                     │
│  pod/net-kourier-controller  (xDS設定をgatewayに配信)│
│  pod/controller              (KnativeリソースをReconcile)│
│  pod/activator               (スケールゼロ時のリクエスト受付)│
│  pod/autoscaler              (Pod数を制御)  ←── HPA │
│  pod/webhook                 (バリデーション) ←── HPA│
│                                                     │
│  ConfigMap/config-network                           │
│    ingress-class: kourier.ingress.networking.knative.dev│
└─────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────┐
│ namespace: default                                  │
│                                                     │
│  Knative Service (ksvc/myapp)                       │
│    URL: http://myapp.default.136.110.97.162.sslip.io│
│    READY: True                                      │
│       │                                             │
│       ├── Configuration/myapp                       │
│       │     latestReady: myapp-00002                │
│       │                                             │
│       ├── Route/myapp                               │
│       │     → myapp-00002 (100%)                    │
│       │                                             │
│       └── Revision                                  │
│             ├── myapp-00001 (False / ContainerMissing) ※gcr.io認証エラーで失敗│
│             └── myapp-00002 (True / nginx:alpine)   │
│                   └── Pod (スケールゼロ中: 0/0)      │
└─────────────────────────────────────────────────────┘
```

## リクエストフロー

```
curl http://myapp.default.136.110.97.162.sslip.io
  │
  ▼
svc/kourier (LoadBalancer) @ kourier-system
  │
  ▼
pod/3scale-kourier-gateway @ kourier-system
  │  (xDS設定はnet-kourier-controllerから受け取る)
  │
  ├─ Podが起動中の場合 ──────────────────────→ pod/myapp @ default
  │
  └─ Podがゼロの場合 → pod/activator → Podを起動 → pod/myapp @ default
```

## スケールtoゼロの動作

- アイドル状態が続くとPodは0にスケールダウン
- リクエストが来るとactivatorがバッファリングしてPodを起動
- 現在: `myapp-00002` は ACTUAL_REPLICAS=0（スケールゼロ中）

## ksvcが自動生成するKubernetes Service

```bash
$ kubectl get svc -n default
NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP                                         PORT(S)                                     AGE
kubernetes            ClusterIP      34.118.224.1     <none>                                              443/TCP                                     30m
myapp                 ExternalName   <none>           kourier-internal.kourier-system.svc.cluster.local   80/TCP                                      5m53s
myapp-00002           ClusterIP      34.118.235.28    <none>                                              80/TCP,443/TCP                              6m7s
myapp-00002-private   ClusterIP      34.118.235.127   <none>                                              80/TCP,443/TCP,9090/TCP,9091/TCP,8012/TCP   6m8s
```

| Service名 | Type | 役割 |
|---|---|---|
| `myapp` | ExternalName | Kourierへの転送エントリポイント |
| `myapp-00002` | ClusterIP | Revision単位のPodへのアクセス |
| `myapp-00002-private` | ClusterIP | activator経由のアクセス（メトリクスポート含む）|

ksvcを1つ作るだけでこれらが自動生成される。

---

## Knative Servingの4リソース

| リソース | 役割 |
|---|---|
| **Service (ksvc)** | 全体を管理するトップレベルリソース |
| **Configuration** | テンプレート（どのイメージを使うか） |
| **Revision** | Configurationの不変なスナップショット |
| **Route** | URLとRevisionのトラフィック割り当て |
