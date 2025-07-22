
# Helm Deployment & Diagnostic Guide for Spring Boot App

## 第一步：驗證應用架構

* `Dockerfile` 可正確建置映像檔，程式碼路徑、Base image與相依套件可以正常拉取

```bash
cd docker/spring-hello-problem/
docker build -t ghcr.io/tisamwillaim/spring-hello-problem:1.0.0 -f Dockerfile .
```

* 確認是否引用外部服務、憑證或相依元件

## 第二步：Helm Template 渲染驗證

使用 Helm 執行 dry-run 測試：

```bash
helm template charts -f charts/values.yaml
helm lint charts
```

* 無 YAML 語法或渲染錯誤
* 所有 Kubernetes 資源皆有正確產出，參數帶入無誤
* 映像、環境變數、埠號、資源限制等皆正確套用

## 第三步：部署至叢集

部署應用至 Kubernetes：

```bash
helm install spring-problem charts/java-app -n default -f ./charts/java-app/values.yaml
```

驗證執行時行為：

* Pod 的 readiness 與 liveness probe 檢查通過

* Volume 掛載正常、檔案系統權限可用

  ```bash
  kubectl exec <pod-name> -- touch /java-dump/test-file
  ```

* `/hello` API 回應正常：

  ```bash
  kubectl exec <pod-name> -- curl localhost:8080/hello
  ```

* Service 與 Ingress 有正確暴露應用：

  ```bash
  kubectl get svc
  kubectl get ingress
  curl http://<cluster-ip>:80/hello
  curl https://<ingress-fqdn>/hello
  ```

## 第四步：異常處理與觀測

當服務無回應時：

* Readiness probe 將會失敗
* PreStop hook 會被觸發並執行 thread dump 收集（亦可手動刪除 Pod 以驗證）：

  ```yaml
  lifecycle:
    preStop:
      exec:
        command:
          - /bin/sh
          - -c
          - |
            PID=$(jps | awk '/.jar/ {print $1}')
            jstack $PID > /java-dump/$(hostname)-$(date +%s).log
  ```

確認觀測系統是否正常運作：

* 可透過 Prometheus JMX Exporter 收集 JVM 指標：

  ```bash
  kubectl exec <pod-name> -- curl localhost:9010/metrics
  ```

* Thread dump 已寫入至 `/java-dump` volume，並可檢視：

  ```bash
  kubectl exec <pod-name> -- ls /java-dump/
  kubectl exec <pod-name> -- cat /java-dump/<dump-file-name>
  ```

* 告警系統或日誌收集器應能反映 probe 失敗與 thread block 的行為
