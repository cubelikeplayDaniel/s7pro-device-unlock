# S7 Pro Device Unlock & Developer Mode Toolkit
### 网易有道词典笔 S7 Pro 开发者模式激活与调试解锁工具箱

本项目专为网易有道词典笔 **S7 Pro (Y08_CV1813_V0 / 平头哥 CV1813H)** 提供系统调试通道配置、免密 ADB 授权、无线 SSH 持久化与开发者环境启用方案。

为保证工程结构清晰，本仓库专注于**设备底层调试接口开放与权限固化**；上层 Falcon MiniApp 开发规范、API 手册与构建 SDK 详见配套主仓库：
👉 **[youdao-dictpen-s7pro-guide (S7 Pro 开发者全栈手册与 SDK)](https://github.com/cubelikeplayDaniel/youdao-dictpen-s7pro-guide)**

---

## 🔍 系统底层调试机制解析

S7 Pro 采用基于 Linux 5.10 的嵌入式 Buildroot 系统，根文件系统挂载于只读压缩镜像（EROFS）：

1. **出厂默认状态**：
   - SSH 守护进程（OpenSSH / dropbear）默认未启动，TCP 22 端口关闭；
   - 根文件系统 `/etc/shadow` 密码经过加密散列保护；
   - ADB 调试脚本 `/usr/bin/adb_auth.sh` 需严格校验鉴权标志 `/tmp/.adb_auth_verified`。
2. **系统原生开机测试钩子**：
   系统启动脚本 `/etc/init.d/S99_run_test_scripts` 在启动链末端保留了工厂自检逻辑：
   ```sh
   [ -f /userdisk/skip_re/skip_login.sh ] || exit 0
   start() {
       /userdisk/skip_re/skip_login.sh &
   }
   ```
   若 `/userdisk/skip_re/skip_login.sh` 存在，内核将以 **UID 0 (root)** 身份自动拉起该脚本。
3. **只读保护绕过（Bind Mount 机制）**：
   通过 Linux 的 `mount --bind` 特性，脚本可将可写分区的文件覆盖到只读根分区之上：
   - 覆盖 `/etc/shadow`：将 root 登录密码设置为固定已知密码；
   - 覆盖 `/usr/bin/adb_auth.sh`：实现 ADB 免密直通调试；
   - 启动系统自带的 `/usr/bin/sshd_service`，常驻监听 `0.0.0.0:22`。

---

## 🚀 调试模式启用方案

本工具箱提供两种激活途径：

### 方案 A：USB 物理文件投递法（推荐，零配置极速生效）

适合绝大多数出厂批次（USB 连接时 MTP 默认放行 `/userdisk/skip_re` 目录）。

#### 步骤：
1. **连接电脑**：使用 Type-C 数据线将词典笔连接至电脑；
2. **复制启动脚本**：
   - 在电脑的文件管理器中打开词典笔存储，进入 `skip_re` 文件夹；
   - 将本仓库 `payload/` 目录下的三个文件复制进去：
     * `skip_login.sh`（开机自动执行的主启动脚本）
     * `shadow`（预置 root 密码表，密码明文：`root`）
     * `adb_auth.sh`（ADB 直通鉴权脚本）
3. **重启设备**：长按电源键重启词典笔；
4. **验证连接**：
   - 设备开机后连接局域网 Wi-Fi；
   - 在电脑终端执行：
     ```bash
     # 无线 SSH 登录 (默认密码: root)
     ssh root@<词典笔IP>

     # 或通过 USB 连接执行 ADB
     adb shell
     ```
   - 成功获取 `#` 提示符（UID 0 root shell），调试模式已永久生效！

---

### 方案 B：局域网应用商店透明代理法（高级备选）

若设备固件批次在 MTP 中隐藏了 `skip_re` 目录，可通过局域网代理劫持应用商店下发配置工具。

#### 步骤：
1. **准备热点与透明代理**：
   - 宿主机（macOS/Linux）开启 Wi-Fi 热点；
   - 运行 `certs/appstore_cert_gen.sh` 生成自签名 SSL 调试证书；
   - 配置端口重定向（将发往 80 与 443 的流量转发至本地代理端口）。
2. **运行代理服务**：
   ```bash
   python3 tools/appstore_server.py
   ```
3. **应用商店安装**：
   - 词典笔连接该热点，在系统自带的“应用商店”中刷新；
   - 点击列表第一项由本地代理注入的“开发者工具箱”完成安装；
   - 打开工具箱，Native 模块将自动创建 `/tmp/.adb_auth_verified` 并写入持久化脚本。

---

## 📦 仓库目录结构

```text
s7pro-device-unlock/
├── payload/
│   ├── skip_login.sh        # 开机自启调试脚本 (拉起 SSH 与直通鉴权)
│   ├── shadow               # root 密码表 (密码统一为 root)
│   └── adb_auth.sh          # ADB 鉴权直通脚本
├── tools/
│   ├── appstore_server.py   # 应用商店透明代理与元数据注入服务
│   └── patch_so.py          # Native 动态库符号配置脚本
├── certs/
│   └── appstore_cert_gen.sh # 局域网自签名 SSL 证书生成工具
└── README.md
```

---

## 🔄 设备还原（关闭调试模式）

如需完全恢复出厂纯净状态：
1. 通过 SSH 或 USB 终端执行：
   ```bash
   rm -f /userdisk/skip_re/skip_login.sh /userdisk/skip_re/shadow /userdisk/skip_re/adb_auth.sh
   ```
2. 重启设备后，系统将恢复原生出厂启动链与只读密码表。

---

## 🔗 相关生态项目

* **[youdao-dictpen-s7pro-guide](https://github.com/cubelikeplayDaniel/youdao-dictpen-s7pro-guide)**：S7 Pro MiniApp 架构逆向手册、54 篇底层技术文档与官方 SDK
* **[s7pro-comic-reader](https://github.com/cubelikeplayDaniel/s7pro-comic-reader)**：高性能 474px 硬件点对点双列画卷漫画阅读器

---

## ⚖️ 免责与安全声明

本工具箱仅供学习、嵌入式系统研究与自研第三方应用调试使用。请遵守当地法律法规，切勿用于未授权的商业或恶意用途。
