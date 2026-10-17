# dsh-hello

DeepSeek Harness 与本机 GitHub 联通性的验证仓库 —— 顺便当配置备忘。

## 这是什么

这台机器上「DeepSeek Harness → GitHub」的链路已经打通：能克隆、提交、推送。
本仓库的内容本身就是通过这条链路推上来的。

- 账号：lyl0600
- 远端：git@github.com:lyl0600/dsh-hello.git
- 默认分支：main

## 环境事实（重要）

| 目标 | 状态 | 说明 |
|---|---|---|
| github.com:443 | 不通 | curl 超时，**不要用 https 方式克隆** |
| api.github.com | 通 | REST API 可用 |
| codeload.github.com | 通 | 归档下载可用 |
| raw.githubusercontent.com | 不通 | raw 链接拿不到 |
| github.com:22 (SSH) | 通 | 日常走这个 |
| ssh.github.com:443 | 通 | 22 被封时的备用通道 |

结论：**所有仓库地址都必须用 SSH 形式** `git@github.com:owner/repo.git`。
浏览器能用 github.com 是因为代理只作用于浏览器，终端里没有代理。

## 已安装

- Git 2.55.0 → `C:\Program Files\Git`
  - 官方 winget 源下载被墙，改从 npmmirror 镜像取得安装包：
    `https://registry.npmmirror.com/-/binary/git-for-windows/`
  - 安装包 Authenticode 签名校验通过（Signer: Johannes Schindelin）
  - 自带 OpenSSH 与 Git Credential Manager
- PATH 转发脚本：`C:\Users\Liang\openclaw-bin\git.cmd` → 指向 `C:\Program Files\Git\cmd\git.exe`
  （让已经开着的会话不必重启就能直接敲 `git`）
- GitHub CLI（gh）：**未安装**，镜像下载反复断流；不影响 clone/push，只影响 PR/issue 自动化

## SSH 配置

私钥：`C:\Users\Liang\.ssh\id_ed25519`（ed25519，无口令短语）
公钥：已添加到 GitHub 账号 lyl0600

`C:\Users\Liang\.ssh\config`：

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    # Port 22 works here. If 22 is ever blocked, use the two lines below instead:
    # HostName ssh.github.com
    # Port 443
```

验证：

```powershell
ssh -T git@github.com
# Hi lyl0600! You've successfully authenticated, but GitHub does not provide shell access.
```

## git 身份

```
user.name  = lyl0600
user.email = 343990590+lyl0600@users.noreply.github.com
init.defaultBranch = main
```

## 常用命令

```powershell
# 克隆
git clone git@github.com:lyl0600/dsh-hello.git

# 日常提交推送
git add -A
git commit -m "feat: something"
git push

# 拉取更新
git pull --rebase

# 分支
git switch -c feature/x
git push -u origin feature/x
```

## 踩过的坑

1. **`.ssh/config` 不能带 UTF-8 BOM**
   PowerShell 5.1 的 `Set-Content -Encoding UTF8` 会写 BOM，OpenSSH 报
   `Bad configuration option: \357\273\277host`。要用无 BOM 写入：

   ```powershell
   [System.IO.File]::WriteAllText($path, $text, (New-Object System.Text.UTF8Encoding($false)))
   ```

2. **DSH 沙箱在本机不可用（已定性：与杀软无关）**
   `workspace-write` 模式下任何 `pwsh` 命令都以 `0xC0000142`（`STATUS_DLL_INIT_FAILED`）退出，
   只能用 `danger-full-access` 执行，因此每条命令都会弹审批。

   排查过程（逐层排除）：

   | 假设 | 结论 | 依据 |
   |---|---|---|
   | 卡巴斯基拦截 | 排除 | 暂停卡巴保护后现象不变，仍 `0xC0000142` |
   | 崩溃留痕 | 无 | 事件日志里没有任何 `pwsh.exe` 的 WER / Application Error 记录（进程在 DLL 初始化阶段就死掉，还没进入异常上报） |
   | 受限令牌与 Win11 不兼容 | 排除 | 用系统 SAFER 受限令牌在 DSH 之外启动 `runas /trustlevel:0x20000 cmd.exe` 和 `powershell.exe` 均正常 |
   | DSH 令牌构造 | 最可能 | 工作区 ACL 中存在合成身份 `S-1-4-40151289-676015424` 的授权项，说明它以「合成 SID + 受限令牌」启动子进程 |

   结论：问题在 DSH 的沙箱实现侧，与本机的卡巴斯基 / 火绒（`usysdiag.exe`）无关，
   应作为 DSH 的 bug 反馈。

3. **https 克隆一律超时**
   即使 GitHub 页面能打开，终端里的 https 克隆也会卡死 —— 这是 SNI/IP 层面的阻断，不是凭证问题。

## 待办

- [ ] 申请 PAT（勾 `repo`），补上建仓库 / 开 PR / 读 issue / 看 Actions 日志等能力
- [ ] 给私钥加 passphrase + Windows ssh-agent，或对单个仓库改用 Deploy Key
- [ ] 如需 GitHub CLI，换镜像重试安装
- [ ] 把沙箱 `0xC0000142` 的排查结论反馈给 DSH

## 安全提示

私钥无口令短语，持有该文件即可推送本账号所有有写权限的仓库。
若机器多人共用，请优先做上面的「待办」第二条。