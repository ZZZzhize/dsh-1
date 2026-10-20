# 工作区环境说明 / Workspace Environment Notes

本文件记录本工作区（D:\dsh-work1）已完成的连接与环境配置，方便后续会话复现。

## 1. Git 仓库连接

| 项目 | 值 |
| --- | --- |
| 工作区 | `D:\dsh-work1` |
| 远程仓库 | `https://github.com/ZZZzhize/dsh-1.git` |
| 当前分支 | `main`（跟踪 `origin/main`） |
| 初始提交 | `2b15fde Initial commit`（仅含 `README.md`，内容为 `# dsh-1`） |

推送权限已实测通过（建测试分支 → push → 删除远程分支 → 本地分支已清理，仓库恢复干净状态）。

## 2. 上游源码

- 上游官方仓库已克隆到 `D:\dsh-upstream`（`https://github.com/deepseek-ai/deepseek-harness.git`，分支 `master`）。
- 这才是 DSH 真正的源码所在：`packages/`（各 `@deepseek-ai/dsh-*` 工作区）、`apps/`（cli / desktop / desktop-host / web）、`docs/`、`scripts/`。
- 开发前必读：`D:\dsh-upstream\AGENTS.md`、`docs/architecture.md`、`docs/AGENTS.md`。
- 包管理器：`pnpm@11.28.5`（见根 `package.json` 的 `packageManager` 字段）。

## 3. TLS 拦截问题的根因与修复（重要）

### 现象
```
fatal: unable to access 'https://github.com/ZZZzhize/dsh-1/':
schannel: next InitializeSecurityContext failed: CRYPT_E_NO_REVOCATION_CHECK (0x80092012)
```

### 根因
本机存在 **TLS 中间人代理**（Clash Verge / Steam++ 一类）。用 openssl 探测 `github.com:443` 得到：

```
issuer = CN=SteamTools Certificate, OU=Technical Department, O=BeyondDimension, C=CN
subject = CN=github.com
```

代理签发的是自签根证书 `C=CN, O=BeyondDimension, OU=Technical Department, CN=SteamTools Certificate`，
指纹 `34064270A4CDC900B0BA654D5925483AB19A073B`，已安装在 Windows 证书库（CurrentUser\Root 与 LocalMachine\Root 各一份）。

因此：
- **curl / Node / 浏览器**正常 —— 它们走 Windows 证书库，能验证该自签根。
- **git（schannel 后端）**失败 —— schannel 去做 CRL 吊销检查，代理环境里 CRL 端点不可达（`crl.digicert.com` 解析失败），报 `CRYPT_E_NO_REVOCATION_CHECK`。
- **git（openssl 后端）**也失败 —— openssl 用的 CA bundle 里没有那张自签根，报 `unable to get local issuer certificate (20)`。

### 修复方式（已生效，验证通过）
1. 从 Windows 证书库导出代理根证书为 PEM：
   `C:\Users\ZZZzhize1\.dsh\git-certs\proxy-root.pem`
2. 与 git 自带 CA bundle 合并：
   `C:\Users\ZZZzhize1\.dsh\git-certs\combined-ca.crt`（121 + 1 = 122 张证书）
3. 写入全局 git 配置（`C:\Users\ZZZzhize1\.gitconfig`）：

```ini
[http]
	sslBackend = openssl
	sslCAInfo = C:/Users/ZZZzhize1/.dsh/git-certs/combined-ca.crt
```

现在 git **完整校验证书链**（不是 `sslVerify=false` 这种降级方案），fetch / push 均正常。

> 注意：`http.sslNoRevoke` / `http.schannelCheckRevoke` 在此环境下**无效**——问题不在开关，而在 CRL 端点不可达。
> 备用 Git 安装在 `D:\git\Git\cmd\git.exe`（2.55.0），它的 schannel 路径能工作，但 DSH 自带的 git（2.56.0）不行；两者现在都已验证。

## 5. 工具链

| 工具 | 位置 / 版本 |
| --- | --- |
| Node | `D:\nodes\node.exe` v24.21.0 |
| pnpm | `C:\Users\ZZZzhize1\bin\pnpm` → 包装 DSH 自带的 `D:\dsh\resources\runtime\primary-runtime\dependencies\pnpm\bin\pnpm.cjs`，版本 11.28.5（与上游 `packageManager` 一致） |
| corepack | v0.36.0（备用 pnpm 12.14.0 已装到 `C:\Users\ZZZzhize1\.dsh\corepack`，非默认） |
| git | DSH 自带 2.56.0（默认 PATH）；另有 `D:\git\Git` 2.55.0 |
| git-lfs | 3.8.0 |

`corepack enable` 无法写入 `D:\nodes`（EPERM），因此用 wrapper 脚本代替。

## 6. 已安装的 DSH 桌面运行时（参考）

- 桌面应用：`D:\dsh\DeepSeek Harness.exe`，版本 `0.2.1-rc.2`
- 打包后的 app.asar：`D:\dsh\resources\app.asar`（18309 个条目；可用 node 脚本按 asar header 偏移直接读文件）
- 运行时清单：`D:\dsh\resources\runtime\primary-runtime\runtime.json`
  - node 24.21.0 / pnpm 11.28.5 / python 3.12.14
  - payloadDigest `e2cef445c262ce6e8d28480b5ebecc8b0c54c06bea7183374debfcf02f30eb7d`
- 用户 profile：`C:\Users\ZZZzhize1\.dsh\profiles\desktop\`（`package.json` / `cordis.yml` / `cordis.patch.yml` / `pnpm-workspace.yaml`）
  - 已装 bundle：`@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`、`dsh-whale-widget`、`dshmarket`、`dsh-better-sidebar`、`dsh-our-free-model`、`dsh-ops`、`billion-context`、`dsh-orb`、`dsh-session-manager`
- Web GUI 当前地址：`http://127.0.0.1:59469`
