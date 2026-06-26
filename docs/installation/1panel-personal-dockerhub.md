# 1Panel 使用个人 DockerHub 镜像

本文档说明如何使用本仓库的 GitHub Actions 流水线构建个人 DockerHub 镜像，并在 1Panel 中将现有 New API 容器替换为自己的最新构建。

## 一、准备 DockerHub 凭据

1. 在 DockerHub 创建或确认可推送的仓库，例如：

```text
your-dockerhub-name/new-api
```

2. 在 DockerHub 创建 Access Token，权限至少需要允许推送镜像。

3. 在当前 GitHub 仓库中进入 **Settings > Secrets and variables > Actions**，添加：

| 类型 | 名称 | 示例 | 说明 |
| --- | --- | --- | --- |
| Variable | `DOCKERHUB_USERNAME` | `your-dockerhub-name` | DockerHub 用户名 |
| Secret | `DOCKERHUB_TOKEN` | `dckr_pat_xxx` | DockerHub Access Token |
| Variable（可选） | `DOCKERHUB_IMAGE` | `your-dockerhub-name/new-api` | 自定义镜像名，不填时默认使用 `DOCKERHUB_USERNAME/new-api` |

`DOCKERHUB_USERNAME` 建议使用 Repository Variable，便于流水线在多架构构建和 manifest 合并阶段复用镜像名；`DOCKERHUB_TOKEN` 必须使用 Secret。

## 二、运行构建流水线

流水线文件位于：

```text
.github/workflows/dockerhub-personal.yml
```

触发方式：

- 推送到 `master` 分支时自动构建并推送。
- Pull Request / Merge Request 合并进入 `master` 后也会触发，因为合并会产生一次到 `master` 的 push。
- 在 GitHub **Actions > Publish personal DockerHub image > Run workflow** 手动触发。

建议的长期分支流程：

```text
upstream/original project -> main -> master -> feature branch -> PR/MR -> master -> DockerHub image
```

- `main` 用来同步原始项目更新，不触发个人镜像构建。
- `master` 是个人稳定发布分支，只要有更新进入 `master`，就构建并推送个人 DockerHub 镜像。
- 日常开发从 `master` 拉出功能分支，开发完成并验证通过后，通过 PR/MR 合并回 `master`。
- 如果原始项目有更新，先把更新拉到 `main`，再把 `main` 合并到 `master`；合并后的 `master` 会自动发布新镜像。

默认会发布多架构镜像：

```text
linux/amd64,linux/arm64
```

流水线会分别使用 amd64 和 arm64 原生 GitHub Actions runner 构建，再合并成多架构 manifest，避免在 x86 runner 上用 QEMU 仿真 arm64 导致构建过慢。

## 三、镜像标签

每次成功构建都会推送这些 tag：

```text
your-dockerhub-name/new-api:latest
your-dockerhub-name/new-api:master
your-dockerhub-name/new-api:sha-<commit-sha>
your-dockerhub-name/new-api:master-<yyyymmdd>-<commit-sha>
```

手动运行时如果填写了 `tag`，还会发布：

```text
your-dockerhub-name/new-api:<tag>
```

1Panel 日常更新建议使用：

```text
your-dockerhub-name/new-api:latest
```

需要回滚时可以换成某个 `sha-*` 或日期版本 tag。

## 四、在 1Panel 中替换镜像

如果已经通过 1Panel 部署过官方镜像，只需要保留原来的数据卷和环境变量，把应用镜像从：

```text
calciumion/new-api:latest
```

改为：

```text
your-dockerhub-name/new-api:latest
```

然后在 1Panel 中重新拉取镜像并重建容器。

使用 Compose 时，核心配置类似：

```yaml
services:
  new-api:
    image: your-dockerhub-name/new-api:latest
    container_name: new-api
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - ./data:/data
      - ./logs:/app/logs
    environment:
      - TZ=Asia/Shanghai
      - SESSION_SECRET=your_session_secret_here
```

如果在服务器终端操作，可以执行：

```bash
docker pull your-dockerhub-name/new-api:latest
docker compose pull new-api
docker compose up -d new-api
```

更新镜像不会自动迁移或删除数据；只要 `./data:/data` 等挂载保持不变，容器重建后仍会使用原来的数据。
