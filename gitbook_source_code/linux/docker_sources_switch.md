# Docker Sources Switching

国内使用 docker 经常会遇到网络限制或者网络失败的情况, 所以需要

## Docker 更换国内镜像源

对于 Ubuntu 系统

`sudo vim /etc/docker/daemon.json` 替换其中内容如下

```json
{
  "data-root": "/docker2/",
  "registry-mirrors": [
    "https://ccr.ccs.tencentyun.com",
    "https://docker.rainbond.cc",
    "https://elastic.m.daocloud.io",
    "https://elastic.m.daocloud.io",
    "https://docker.m.daocloud.io",
    "https://gcr.m.daocloud.io",
    "https://ghcr.m.daocloud.io",
    "https://k8s-gcr.m.daocloud.io",
    "https://k8s.m.daocloud.io",
    "https://mcr.m.daocloud.io",
    "https://nvcr.m.daocloud.io",
    "https://quay.m.daocloud.io"
  ],
  "insecure-registries": ["0.0.0.0/0"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

再重启docker服务, 启用镜像源, 再 `docker pull xxx` 应该就可以拉取目标镜像.

