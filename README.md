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
| `cleanup` | | `none` | `none` = 不碰已有包；`purge` = 推之前把整个包删掉再推（历史 tag 全清）；`prune` = 推完只保留最近 `keep_versions` 个 tag |
| `keep_versions` | | `5` | `cleanup=prune` 时保留的 **tag** 数（按 tag 整组保留，不会拆散多架构镜像） |

示例：`source=nginx:1.27`、`target=my-nginx`、`tags=latest,1.27`、`platforms=all`
→ 产出 `ghcr.io/<你>/my-nginx:latest` 和 `:1.27`，两个 tag 都是完整多架构。

### 3. 拉取

```bash
docker pull ghcr.io/<你>/my-nginx:latest

# 国内加速（南京大学镜像站）
docker pull ghcr.nju.edu.cn/<你>/my-nginx:latest
```

运行结束后 Job Summary 会列出每个 tag 的平台清单、digest 和拉取命令，复制即可。

## 清理已经推上去的镜像

GitHub **没有批量删除包的入口**：只能在 `https://github.com/users/<你>/packages/container/<包名>/settings`
里逐个点 Delete，包一多很烦。所以另带一个工作流 **清空 GHCR 包**（`purge-packages.yml`）。

### 先配一个 token（必须）

删包只能用 **PAT**：`GITHUB_TOKEN` 是 GitHub App 的安装令牌，访问 `/user/*` 一律
`403 Resource not accessible by integration`，而删包 API 只有 `/user/packages` 和
`/orgs/<org>/packages` 两条路。

Settings → Developer settings → **Personal access tokens (classic)** → 勾
`read:packages` + `delete:packages`（写代码仓库的话再加 `repo`）→ 生成后存到
仓库 **Settings → Secrets and variables → Actions → `PACKAGES_TOKEN`**。

### 参数

Actions → **清空 GHCR 包** → Run workflow：

| 参数 | 默认 | 说明 |
|---|---|---|
| `scope` | `repo` | `repo` = 只删关联到本仓库的包；`all` = 账号/组织名下**所有**容器包 |
| `packages` | 空 | 指定包名（逗号分隔），填了就忽略 `scope` |
| `keep_versions` | `0` | `0` = 整包删除；>0 = 每个包只留最近 N 个 tag |
| `dry_run` | `true` | 默认**只预览**，打印"会删什么"和版本数 |
| `confirm` | 空 | 关掉 dry_run 后必须填 `DELETE`，否则直接报错退出 |

跑完 Job Summary 有一张表：包名 / 版本数 / 动作 / 结果，最后一栏是复核剩余包。

### 不想开 Actions 时

```bash
curl -X DELETE -H "Authorization: Bearer $GH_TOKEN" \
     -H "Accept: application/vnd.github+json" \
     https://api.github.com/user/packages/container/<包名>
```

组织下的包路径换成 `/orgs/<org>/packages/container/<包名>`；只删某个 tag 用
`/user/packages/container/<包名>/versions/<version_id>`（先 `GET .../versions` 拿 id）。

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
- `cleanup=purge` 是"推倒重来"，会连历史 tag 一起没了；只想控制版本数量用 `prune`。
- `prune` 按 **tag 分组**：一个多架构 tag 在 GitHub 上占 1 个 index + N 个子 manifest
  version（busybox 这种小镜像就有 18 个），按"version 个数"保留会把同一个镜像的兄弟架构删掉，
  拉下来直接 404。删除旧 tag 时会把它的 index 和子 manifest 一起删，同时跳过仍被保留 tag
  引用的子 digest（不同 tag 常共享同一平台镜像）。
- 单个 job 上限 90 分钟；超大镜像建议自托管 runner。
- 输入全部经环境变量传递，镜像名不会被当成 shell 命令执行。
- 目标命名空间取仓库所有者（`github.repository_owner`），协作者触发也不会推到自己名下。

## 与旧版差异

| 旧文件 | 现状 |
|---|---|
| `.github/workflows/main.yml` | 合并进 `mirror.yml` |
| `.github/workflows/mailplatform.yml` | 合并进 `mirror.yml`（用 `platforms` 参数），文件名笔误一并去掉 |
