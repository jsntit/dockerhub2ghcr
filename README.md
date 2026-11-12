# 一键把 DockerHub 镜像搬运到 GitHub 容器注册表 (GHCR.IO)

## 项目简介

通过 GitHub Actions 把 DockerHub 镜像**整份**复制到自己的 GHCR，避免镜像被撤回、封禁或限流。
公有仓库的 Actions 不计时长（私有仓库每月 2000 分钟），配合 **南京大学开源镜像站** 还能加速国内拉取。

实现上用 **skopeo** 做 registry → registry 直传：

- 不落地、不需要 dockerd，几 GB 的镜像也只用 runner 很少的磁盘；
- **保留完整 manifest list**，amd64 / arm64 / armv7 一起搬过去，ARM 设备拉下来不会
  `exec format error`（旧版 `docker pull` + `docker tag` + `docker push` 会丢多架构，见下文）。

## 为什么旧做法会丢架构

| 步骤 | 旧版（docker pull/tag/push） | 现在（skopeo copy --all） |
|---|---|---|
| 拉取 | 只拉 runner 所在架构（amd64） | 只读 manifest，按需搬 blob |
| 推送 | 推上去是**单平台 manifest**，原 index 被覆盖 | 原样搬 manifest list |
| 结果 | ARM 设备拉不到、或拉到 amd64 跑不起来 | 各架构都能拉 |
| 磁盘 | pull + tag + push 双份占用 | 不落盘 |

## 使用步骤

### 1. 把仓库导入到自己的账号

打开 [GitHub 导入页面](https://github.com/new/import)，"Your old repository's clone URL" 填
`https://github.com/jsntit/dockerhub2ghcr.git`，按提示导入。

> 直接 Fork 也能跑，但 Actions 用量记在 fork 仓库上，且不会自动跟随上游更新；
> 导入成独立仓库更清爽。

### 2. 运行工作流

Actions → **镜像搬运 DockerHub → GHCR** → Run workflow，填参数：

| 参数 | 必填 | 默认 | 说明 |
|---|---|---|---|
| `source` | ✅ | `nginx:latest` | 源镜像。`docker.io` 和 `library/` 可省略：`nginx`、`alpine:3.20`、`library/python:3.12`；也支持 `nginx@sha256:...` |
| `target` | | 同源镜像名 | 推到 GHCR 的镜像名，不含命名空间，可写 `org/name` |
| `tags` | | 沿用源 tag | 目标 tag，逗号分隔多个：`latest,1.27`；用 `@sha256` 引用源镜像时**必填** |
| `platforms` | | `all` | `all` = 整份多架构；或 `linux/amd64`、`linux/arm64,linux/amd64` 只搬指定平台 |
| `namespace` | | 仓库所有者 | GHCR 命名空间（用户或组织） |

示例：`source=nginx:1.27`、`target=my-nginx`、`tags=latest,1.27`、`platforms=all`
→ 产出 `ghcr.io/<你>/my-nginx:latest` 和 `:1.27`，两个 tag 都是完整多架构。

### 3. 拉取

```bash
docker pull ghcr.io/<你>/my-nginx:latest

# 国内加速（南京大学镜像站）
docker pull ghcr.nju.edu.cn/<你>/my-nginx:latest
```

运行结束后 Job Summary 会列出每个 tag 的平台清单、digest 和拉取命令，复制即可。

## 两个必须知道的点

**① 包默认是私有的。** 首次推送会在你的账号下创建 **private** 包；
南京大学镜像站、其他机器匿名拉取都要求 **public**。改法：
个人 → Settings → Packages（或包详情页）→ Package settings → Change visibility → Public。
组织仓库首次跑还可能要在 Organization settings → Actions → Workflow permissions 里
允许 `GITHUB_TOKEN` 写 packages。

**② 匿名拉 Docker Hub 会被限流。** GitHub runner 是共享 IP，匿名额度 100 次/6 小时，
常见 `toomanyrequests: 429`。仓库 Settings → Secrets 里加上
`DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`（只读 token 即可），工作流会自动登录；
不加也能跑，只是可能偶发失败。

## 其它

- `platforms` 只搬部分架构时，目标是**重建的 manifest**（不含未选架构），tag 不带架构后缀。
- 单个 job 上限 90 分钟；超大镜像建议自托管 runner。
- 输入全部经环境变量传递，镜像名不会被当成 shell 命令执行。
- 目标命名空间取仓库所有者（`github.repository_owner`），协作者触发也不会推到自己名下。

## 与旧版差异

| 旧文件 | 现状 |
|---|---|
| `.github/workflows/main.yml` | 合并进 `mirror.yml` |
| `.github/workflows/mailplatform.yml` | 合并进 `mirror.yml`（用 `platforms` 参数），文件名笔误一并去掉 |
