## 設計決策

- 設定 terminationGracePeriodSeconds，確保服務在終止前有足夠時間處理尚未完成的請求。並且可以根據環境調整，避免測試環境需要等待太久。
- readinessProbe 設定較短的間隔，用於更快偵測應用異常；livenessProbe 間隔較長，避免頻繁檢查造成 Kubernetes 不必要的負載。
- 使用 PersistentVolume 搭配 PersistentVolumeClaim，而非直接掛載 hostPath，讓儲存空間的使用更清楚、易於管理。

## 假設條件

- 環境中的安全控管較寬鬆，容器可直接使用 privileged 權限，無需自訂 ServiceAccount 或在 Dockerfile 設定特定 UID。
- 集群中已部署 Prometheus 與 Alertmanager，可處理監控與告警需求。
- 此部署為測試用途，因此未特別考慮在高負載下 CPU Throttling 等效能調教。

## 建議與改進

- jstack 產生的 thread dump 可儲存至掛載的 NFS 卷，避免節點異常時無法存取。或是異常發生時不斷產生dump檔案，造成系統硬碟空間不足
- readiness 與 liveness 分別使用不同檢查端點。前者反映服務是否可對外提供功能，後者則檢查系統是否卡死，需重啟 Pod 才能恢復。也可以分別使用Layer 7和Layer 4的Health Check，近一步減少kubelet的資源消耗
- jvmExporter.enabled 的必要性可視環境而定。在DR環境外，多數情境都需要監控，因此可考慮將 sidecar 模式改為使用 Java Agent 的方式嵌入主容器，減少額外資源消耗。