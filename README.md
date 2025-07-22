
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
helm template charts -f charts/java-app/values.yaml
helm lint charts
```

* 無 YAML 語法或渲染錯誤
* 所有 Kubernetes 資源皆有正確產出，參數帶入無誤
* 映像、環境變數、埠號、資源限制等皆正確套用

## 第三步：部署至叢集

部署應用至 Kubernetes：

```bash
helm install spring-problem charts -n default -f charts/java-app/values.yaml
```

驗證執行時行為：

* Pod 的 readiness 與 liveness probe 檢查通過

* Volume 掛載正常、檔案系統權限可用

  ```bash
  kubectl exec <pod-name> -- sh -c "touch /java-dump/test-file; ls /java-dump"
  ```

* `/hello` API 回應正常：

  ```bash
  kubectl exec <pod-name> -- curl localhost:8080/hello
  ```

* Service 與 Ingress 有正確暴露應用：

  ```bash
  curl http://<cluster-ip>:80/hello
  curl https://<ingress-fqdn>/hello
  ```

* JMX有正常運作：

  ```bash
  kubectl exec <pod-name> -- curl localhost:9010/metrics
  ```

## 第四步：異常處理與觀測

當服務無回應時：

* Readiness probe 將會失敗，異常Pod的服務會中斷，並透過Liveness Probe重啟節點
* PreStop hook 會被觸發並執行 thread dump 收集，可以連線到節點內部搜集，或是使用另外一個Pod掛載共用相同PVC，將Dump取出
* Prometheus中的metrics透過prometheusrule發出告警
