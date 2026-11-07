# tailnet-go

`package mobile`：用 gomobile 把 tsnet 暴露给 Android。

导出：`Start(hostname, authKey, stateDir, controlURL)`、`Stop()`、`StatusJSON()`、`Forward(localPort, host, port)`、`CloseForward(...)`。

编译：

```sh
cd go
go mod download
gomobile bind -target=android/arm64 -androidapi 26 -o ../android/tailnet.aar .
```

所有注释保持 ASCII：gobind 会把注释抄进生成的 Java，中文注释在 zh-CN 的 javac 上按 GBK 读会炸（这是个真踩过的坑）。
