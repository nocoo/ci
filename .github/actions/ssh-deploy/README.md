# ssh-deploy

通过 SSH 在远程服务器执行部署脚本。零第三方依赖，纯 OpenSSH + bash 实现，作为
`appleboy/ssh-action@v1` 的内部替代品，消除单点供应链风险。

## 用法

```yaml
- name: Deploy to production server
  uses: nocoo/base-ci/.github/actions/ssh-deploy@<sha>  # v2026.5
  with:
    host: ${{ secrets.VPS_HOST }}
    user: ${{ secrets.VPS_USER }}
    port: ${{ secrets.VPS_PORT }}
    key:  ${{ secrets.VPS_SSH_KEY }}
    envs: IMAGE_SHA
    script: |
      set -euo pipefail
      cd /opt/ellie
      echo "${{ secrets.GHCR_PULL_TOKEN }}" | docker login ghcr.io \
        -u "${{ secrets.GHCR_PULL_USER }}" --password-stdin
      docker compose pull web admin
      docker compose up -d --no-deps web admin
  env:
    IMAGE_SHA: ${{ steps.sha.outputs.value }}
```

## Inputs

| input         | required | default | 说明                                                                 |
| ------------- | -------- | ------- | -------------------------------------------------------------------- |
| `host`        | ✅       | —       | SSH 目标主机                                                         |
| `user`        | ✅       | —       | SSH 用户名                                                           |
| `port`        | ❌       | `22`    | SSH 端口                                                             |
| `key`         | ✅       | —       | SSH 私钥（整段 PEM，含 BEGIN/END 行）                                |
| `envs`        | ❌       | `""`    | 空格分隔的变量名列表，从 runner 环境带到远端                         |
| `script`      | ✅       | —       | 在远端执行的 bash 脚本（多行）                                       |
| `known-hosts` | ❌       | `""`    | 预置 host key 列表，留空则现场用 `ssh-keyscan` 抓                    |

## 与 `appleboy/ssh-action@v1` 兼容性对照

| 字段          | appleboy/ssh-action     | ssh-deploy            | 说明                                  |
| ------------- | ----------------------- | --------------------- | ------------------------------------- |
| `host`        | `host`                  | `host`                | 等价                                  |
| `username`    | `username`              | `user`                | **重命名**：迁移时改成 `user:`        |
| `port`        | `port` (default `22`)   | `port` (default `22`) | 等价                                  |
| `key`         | `key`                   | `key`                 | 等价                                  |
| `envs`        | `envs: "A B"`           | `envs: "A B"`         | 等价（空格分隔变量名）                |
| `script`      | `script: \|`            | `script: \|`          | 等价                                  |
| host key 校验 | 默认跳过（`-o StrictHostKeyChecking=no`） | 默认现场 `ssh-keyscan` 写 known_hosts | 行为等价；`known-hosts` 入参可显式预置 |
| 远端环境变量  | 通过 ssh `SendEnv`/server `AcceptEnv`     | 通过 `env VAR=val bash -s` 前缀     | 不依赖 sshd 配置，更可移植             |
| 命令注入      | input 字符串拼接到远端                    | `env:` 块隔离 + here-string 灌 stdin | 更安全，避免 GHA 表达式注入           |

## 安全细节

- 所有敏感 input（`key` / `script` / `envs`）通过 step 的 `env:` 块传递，**绝不**在
  `run:` 里直接 `${{ inputs.x }}` 字符串插值——避免 GHA expression injection。
- 远端 script 用 here-string `<<<` 灌进 ssh 的 stdin，不作为 ssh 命令行参数，避免 shell
  二次解析。
- `envs` 列出的变量值用 `printf '%q'` 安全 quote 后拼到远端命令前缀，可正确处理空格、
  引号、特殊字符。
- `BatchMode=yes` 防止远端 prompt 卡死 runner；`ServerAliveInterval=30` 防长部署被
  中间设备断 idle 连接。
- 私钥文件 `~/.ssh/deploy_key` 在 cleanup step 用 `if: always()` 删除，不会泄露到后续
  step。

## 已知 pitfall

- **secret 值含单引号**：`printf '%q'` 在 bash 里能正确转义（输出 `\'`），但远端必须是
  bash（不是 dash/sh）。GitHub Actions runner 的 ssh 默认调用对端用户 shell，所以远端
  shell 是 bash 时安全。生产 VPS 上的 root/deploy 用户基本都是 bash，实际验证下来
  `VPS_*` / `GHCR_*` 这类 secret 不会触碰这条边界。
- **here-string 缓冲区上限**：`<<< "$DEPLOY_SCRIPT"` 把整个脚本灌 stdin，超长（> 64KB
  量级）可能撞 ssh/管道缓冲区。实测 4KB 内安全，下游目前最长的 ellie release.yml ssh
  脚本约 1.5KB，远在安全区。
- **`envs` 变量必须先在 step 的 `env:` 块设置**：`with: envs: FOO` 只是声明要传递
  `FOO`，`FOO` 本身得在调用方 step 的 `env:` 中赋值，否则 action 会跳过并打 warning。
- **不预置 `known-hosts` 时**：默认行为是现场 `ssh-keyscan` 写 known_hosts，等价
  `StrictHostKeyChecking=no` 的可控版本——能拦后续的 host key 变更，但首次连接没有
  TOFU 校验。生产建议把 `ssh-keyscan -H <host>` 输出存入 GitHub secret，通过
  `known-hosts:` 显式传入。

## 下游迁移指引（不在本 issue 范围内）

下游 `ellie` / `noheir` / `neo` / `wooly` 的 `release.yml` 替换 step 时：

```diff
 - name: Deploy to production server
-  uses: appleboy/ssh-action@v1
+  uses: nocoo/base-ci/.github/actions/ssh-deploy@<sha>  # v2026.5
   env:
     IMAGE_SHA: ${{ steps.sha.outputs.value }}
   with:
     host: ${{ secrets.VPS_HOST }}
-    username: ${{ secrets.VPS_USER }}
+    user: ${{ secrets.VPS_USER }}
     key: ${{ secrets.VPS_SSH_KEY }}
     port: ${{ secrets.VPS_PORT }}
     envs: IMAGE_SHA
     script: |
       ...
```

唯一需要改的是 `username` → `user`，其他 input 名称、`script` 内容、`env:` 用法都保持
原样。
