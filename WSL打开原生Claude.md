# WSL 终端默认运行原生 Claude Code

## 问题

在 Windows 路径下启动、但工作区位于 WSL（Ubuntu）虚拟机中。在 WSL 终端里敲 `claude`，跑的其实是 **Windows 版** 的 claude，而不是 Linux 原生版。

希望：打开 WSL 终端敲 `claude` 时默认运行 WSL 原生的 claude。

## 根因诊断

| 检查项 | 结果 | 说明 |
|--------|------|------|
| WSL 里 `which claude` | `/mnt/c/Users/12983/AppData/Roaming/npm/claude` | 借用了 Windows 版 |
| WSL 是否原生装了 claude | 否 | 所以兜底命中 Windows 那个 |
| WSL 里 `which npm` | `/mnt/c/Program Files/nodejs/npm` | npm 也是 Windows 的 |
| `npm config get prefix` | `C:\...\npm` | 直接 `npm i -g` 会装到 Windows 侧，无效 |
| WSL 原生 node | `/usr/bin/node` v22.22.1（apt 精简版，**不带 npm**） | 有 node 但缺 npm |

**核心原因**：WSL 开启了 PATH interop，把 Windows 的 PATH 追加到末尾（含 `/mnt/c/Users/12983/AppData/Roaming/npm`）。由于 WSL 本身没有原生 claude，`which claude` 兜底找到了 Windows 版。

## 解决过程

### 1. 官方安装脚本走不通（区域封锁）

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

返回的是 "App unavailable in region" 页面（重定向到 `https://claude.com/app-unavailable-in-region`），claude.ai 网站在该区域被地理封锁。**此路不通。**

但 npm registry 可达：

```bash
curl -fsSL -o /dev/null -w "%{http_code}\n" https://registry.npmjs.org/@anthropic-ai/claude-code
# → 200
```

### 2. 用 corepack 提供的原生 npm 安装

WSL 的 Ubuntu node 不带 npm，但自带 `corepack`（可提供 npm 11.17.0，且免 sudo）。把全局前缀设到用户目录 `~/.local`，安装官方包：

```bash
corepack npm config set prefix "$HOME/.local"
corepack npm install -g @anthropic-ai/claude-code
```

> 注意：首次安装时 postinstall 脚本（`node install.cjs`）被 npm 的 allow-scripts 策略拦截未执行，需要重新允许：

```bash
corepack npm install -g --allow-scripts=@anthropic-ai/claude-code @anthropic-ai/claude-code
```

postinstall 会拉取平台二进制，落在：
`~/.local/lib/node_modules/@anthropic-ai/claude-code/node_modules/@anthropic-ai/claude-code-linux-x64/claude`

### 3. 配置 PATH 让原生版优先

在 `~/.bashrc` 末尾追加（保证交互式 shell 都优先于末尾的 Windows interop 路径）：

```bash
# Ensure native (Linux) claude in ~/.local/bin takes precedence over the
# Windows claude exposed via WSL PATH interop.
export PATH="$HOME/.local/bin:$PATH"
```

> `~/.profile` 本就有 `if [ -d "$HOME/.local/bin" ]; then PATH="$HOME/.local/bin:$PATH"; fi`，但只在 login shell 生效，`.bashrc` 这行做兜底。
> 未修改 `/etc/wsl.conf`，**保留 Windows PATH interop**，互不影响。

## 验证

新开 WSL login shell：

```bash
which claude
# → /home/dftuser/.local/bin/claude   ✅（不再是 /mnt/c/...）

which -a claude
# /home/dftuser/.local/bin/claude      ← 原生版优先
# /mnt/c/Users/12983/AppData/Roaming/npm/claude   ← Windows 版退到末尾

claude --version
# → 2.1.183 (Claude Code)   ✅
```

`which claude` 指向 `~/.local/bin/claude` 即说明 WSL 终端默认走原生版本。之后 `cd` 到 WSL 项目目录直接敲 `claude` 即可。

## 已知瑕疵与说明

1. **二进制文件名是 `claude.exe`**：install.cjs 在 Windows interop 环境下误判平台，把 Linux 二进制命名成了 `.exe`。
   - `file` 确认它实为 **Linux ELF 64-bit 可执行文件**（不是 Windows PE），能正常运行，不影响使用。
   - 软链接：`~/.local/bin/claude -> ../lib/node_modules/@anthropic-ai/claude-code/bin/claude.exe`
2. **配置独立**：原生 claude 与 Windows 版各自独立登录/配置，首次运行原生版可能需重新登录一次。
3. **后续升级**：`claude update`。

## 关键命令速查

```bash
# 安装（区域封锁导致官方 install.sh 不可用时的替代方案）
corepack npm config set prefix "$HOME/.local"
corepack npm install -g --allow-scripts=@anthropic-ai/claude-code @anthropic-ai/claude-code

# PATH 兜底（追加到 ~/.bashrc）
export PATH="$HOME/.local/bin:$PATH"

# 验证
which claude && claude --version
```

## 安全提醒（与本任务无关）

`~/.bashrc` 中存在明文 `MP_API_KEY=...`。明文密钥置于 rc 文件有泄露风险，建议挪到不入库的安全位置并轮换该密钥。

---
*记录日期：2026-06-20*
