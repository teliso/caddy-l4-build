# caddy-l4-build

自动构建带 [caddy-l4](https://github.com/mholt/caddy-l4)（Layer 4 TCP/UDP 代理模块）的 [Caddy](https://caddyserver.com/) Docker 镜像，并发布到 GitHub Container Registry。

## 镜像

```
ghcr.io/teliso/caddy-l4
```

| 标签 | 说明 |
|---|---|
| `latest` | 最新构建 |
| `v<Caddy 版本>-l4-<caddy-l4 版本>` | 具体版本，例如 `v2.11.8-l4-v0.1.2` |

只保留最近 3 个版本，更早的版本会被自动删除。需要固定版本时请注意这一点。

镜像基于 `caddy:alpine`，其中的 `caddy` 二进制被替换为带 layer4 模块的构建。

## 使用

```bash
docker pull ghcr.io/teliso/caddy-l4:latest
```

拉取后需要重新创建容器才会使用新镜像。如果使用 Docker Compose：

```bash
docker compose pull
docker compose up -d
```

如果包是私有的，需要先登录：

```bash
docker login ghcr.io
```

### 示例：转发 TCP 端口

`Caddyfile`：

```
{
	layer4 {
		:5432 {
			route {
				proxy 10.0.0.10:5432
			}
		}
	}
}
```

运行：

```bash
docker run -d --name caddy-l4 \
  -p 5432:5432 \
  -v "$PWD/Caddyfile:/etc/caddy/Caddyfile" \
  ghcr.io/teliso/caddy-l4:latest
```

更多配置方式请参考 [caddy-l4 文档](https://github.com/mholt/caddy-l4)。

## 自动构建

工作流 `.github/workflows/build.yml` 每 3 天运行一次，也可以在 Actions 页面手动触发。

1. 通过 Go module proxy 查询 Caddy 和 caddy-l4 的最新版本，与 `versions.env` 中记录的版本比较。
2. 没有变化就结束。手动触发总是会构建。
3. 有变化时，用 xcaddy 构建，验证 layer4 模块存在，然后推送镜像。
4. 成功后把新版本写回 `versions.env`，再删除超出最近 3 个的旧版本。

构建失败时不会推送镜像，也不会记录版本，下一次运行会自动重试。

只有 Caddy 和 caddy-l4 的新版本会触发构建。Go、xcaddy 和基础镜像始终取最新版本（`setup-go` 使用 `stable`，xcaddy 使用 `@latest`，基础镜像为 `caddy:alpine`），在下一次构建时自动生效，不会单独触发构建。

构建时关闭了来源证明（provenance），所以每个标签对应单个镜像清单。

## 文件

| 文件 | 作用 |
|---|---|
| `.github/workflows/build.yml` | 检查、构建、推送、清理的工作流 |
| `versions.env` | 最近一次构建使用的 Caddy 和 caddy-l4 版本，由工作流自动更新 |
| `Dockerfile` | 把构建好的 `caddy` 二进制放进 `caddy:alpine` |

`versions.env` 单独存放版本号，是因为工作流使用的 `GITHUB_TOKEN` 没有权限修改 `.github/workflows/` 下的文件。

## 仓库设置要求

- Settings → Actions → General → Workflow permissions 需要选择 "Read and write permissions"，工作流才能提交 `versions.env`。
- 工作流需要 `packages: write` 权限来推送和清理镜像，已在工作流中声明。
