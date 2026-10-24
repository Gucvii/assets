# TextRule 补充规则

这些文件只包含公开分流规则，不包含订阅、节点、配置或凭据。

| 文件 | 用途 | Mihomo behavior |
| --- | --- | --- |
| `github-extra.txt` | GitHub 共享云端点补充 | `classical` |
| `huggingface-extra.txt` | Hugging Face 域名补充 | `classical` |
| `docker-k8s.txt` | 海外主流容器镜像仓库、认证和镜像层下载端点 | `classical` |
| `wloc-location.txt` | Apple 网络定位（WLOC）服务域名，需由使用方指定 `DIRECT` | `classical` |

所有文件使用 `format: text`。策略组名称由使用方配置，规则文件不写策略。

## WLOC 定位服务

`wloc-location.txt` 覆盖 WLOC 模块 `[MITM] hostname` 中的 5 个定位主机：
`gs-loc.apple.com`、`gs-loc-cn.apple.com`、`gsp-ssl.ls.apple.com`、
`bluedot.is.autonavi.com`、`bluedot.is.autonavi.com.gds.alibabadns.com`。

使用方必须把这些规则指定为 `DIRECT`，并放在所有 Apple、CDN、国内规则之前：

```yaml
rules:
  - RULE-SET,wloc_location,DIRECT
```

原因：

- `REJECT` 不可用。WLOC 的改写发生在 `http-response` 阶段，需要 Apple 的真实响应体；
  掐断连接会使脚本不触发，`locationd` 随后降级到 GPS，结果是显示真实位置。
- `PROXY` 不建议。绕节点增加往返延迟，可能让 `locationd` 超时降级到 GPS。
- 必须优先于 `apple_cdn`、`apple_cn_non_ip`、`apple_services` 和 `domestic_non_ip`。
  `apple_services` 的 `DOMAIN-SUFFIX,apple.com` 会命中 `gsp-ssl.ls.apple.com` 并把流量
  导向可被切换到节点的策略组；显式直连规则可以消除这个漂移。

规则全部为精确 `DOMAIN`，不做后缀匹配，因此不会劫持其他 `apple.com` 或
`autonavi.com` / `alibabadns.com` 子域。

本规则只影响使用方自身的网络路由，不构成对任何服务的绕过或绕过尝试；
定位测试请仅用于自己拥有或已获授权的设备。

## Docker / Kubernetes 镜像仓库

覆盖 Docker Hub / DHI、GHCR、Quay / Red Hat、Kubernetes Registry、Google GCR / Docker Artifact Registry、Microsoft MCR / Azure ACR、Amazon ECR、NVIDIA NGC 和 GitLab.com Registry。

建议使用手动 `Docker` 分组，首选项为 `最终选择`，并将其 `RULE-SET` 放在 GitHub、微软、Google 和通用 CDN 规则之前，使 GHCR 等仓库进入 Docker 分组。

规则覆盖仓库专属域名及已确认的镜像层下载端点，不匹配整个 `amazonaws.com`、`cloudfront.net`、`googleapis.com`、`pkg.dev`、`blob.core.windows.net` 或 `cloudflarestorage.com`。Google Artifact Registry 仅匹配 `*-docker.pkg.dev`；S3 仅匹配 Kubernetes / ECR 的特定桶命名。

国内仓库、自建仓库的自定义域名和普通 GitHub/GitLab 页面不在此规则范围。GitLab 在 `gitlab.com` 上的共享认证路径仍遵循原有 GitLab 分流；域名规则无法按 URL 路径区分。使用旧式共享 Blob 存储的私有 ACR，或厂商更换到未知下载主机时，应根据实际主机名补充，避免扩大到整个云平台。

维护时参考以下官方资料，核对认证、清单与镜像层下载涉及的主机：

- [Docker allowlist](https://docs.docker.com/desktop/enterprise/allow-list/)，以及 [Docker 确认的 R2 下载域名](https://github.com/docker/docs/issues/21960)。
- [GitHub runner 网络要求](https://docs.github.com/en/actions/reference/runners/github-hosted-runners#communication-requirements-for-github-hosted-runners)。
- [Quay / Red Hat 镜像与 CDN](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.3/html/creating_and_consuming_execution_environments/populate-container-registry)。
- [Kubernetes Registry 重定向排查](https://github.com/kubernetes/registry.k8s.io/blob/main/docs/debugging.md)。
- [Google Docker Artifact Registry 命名](https://docs.cloud.google.com/artifact-registry/docs/docker/names)。
- [Azure ACR 端点](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-firewall-rules)，[MCR 数据端点](https://learn.microsoft.com/en-us/azure/aks/outbound-rules-control-egress)。
- [Amazon ECR 端点](https://docs.aws.amazon.com/general/latest/gr/ecr.html)，[ECR 镜像层桶](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html)。
- [NVIDIA NGC 网络协议](https://docs.nvidia.com/ngc/latest/ngc-user-guide.html#nvidia-ngc-network-protocols)。
- [GitLab Container Registry](https://docs.gitlab.com/user/packages/container_registry/)。
