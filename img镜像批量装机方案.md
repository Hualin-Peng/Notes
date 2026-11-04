# img 镜像批量装机方案（无 PXE）

> 场景：多台测试机，需频繁重装系统，**无法使用 PXE**。
> 现有方式：**Rufus 把 img 烧写进 U 盘 → 从 U 盘启动 → 自动格式化 UFS + 无人值守安装**。
> 目标：省掉"每次重烧 U 盘"，并让"开机→安装→重启"尽量自动化。
> 文档日期：2026-11-05

---

## 目录

- [1. 现状与关键结论](#1-现状与关键结论)
- [2. img 和 iso 的本质区别](#2-img-和-iso-的本质区别)
- [3. 先判断你的 img 属于哪一类](#3-先判断你的-img-属于哪一类)
- [4. 路线 A：Ventoy 直接启动 img](#4-路线-aventoy-直接启动-img)
- [5. 路线 B：dd 整盘写入](#5-路线-bdd-整盘写入)
- [6. 通用难题：装完不要循环启动](#6-通用难题装完不要循环启动)
- [7. 网络路径：HTTP Boot / iPXE](#7-网络路径http-boot--ipxe)
- [8. 自动化脚本](#8-自动化脚本)
- [9. 性能预期](#9-性能预期)
- [10. 风险与排查](#10-风险与排查)
- [11. 待确认事项](#11-待确认事项)

---

## 1. 现状与关键结论

### 1.1 你现在的流程

```
Rufus 把 img 逐字节写进 U 盘
        ↓
U 盘启动
        ↓
安装器自动格式化 UFS + 无人值守安装
```

### 1.2 三个关键结论

| # | 结论 | 含义 |
|---|---|---|
| 1 | **你的 img 已自带无人值守能力** | 不需要 `unattend.xml`、不需要 Windows Setup 向导，之前讨论的答案文件方案**全部作废** |
| 2 | **问题只剩两个** | ① 每次都要 Rufus 重烧 U 盘 ② 开机没法自动选 U 盘启动 |
| 3 | **img 自带分区表，只能"写"不能"装"** | `Windows Setup` 的"分区→展开 WIM"流程对它**完全不适用** |

### 1.3 因此方案的目标变成

```
把"Rufus 烧写" → 换成"拷文件"（或 dd）
把"手动选启动项" → 换成"自动"
```

其余（无人值守安装）**保持原样即可**。

---

## 2. img 和 iso 的本质区别

| | ISO | **IMG（整盘镜像）** |
|---|---|---|
| 内容 | 文件系统（ISO9660/UDF） | **完整磁盘结构：MBR/GPT + 分区 + 文件系统 + bootloader** |
| 启动方式 | 作为 CD/DVD 被挂载 | 作为**虚拟硬盘**被识别 |
| 能否直接启动 | ✅ | ✅（前提是 bootloader 完整） |
| 能否"安装" | ✅ 运行安装器 | ❌ **只能整盘写入** |
| 分区 | 安装器决定 | **镜像里已经定死** |

**核心推论**：img 写到哪里，那块盘就变成 img 的样子。所以：

- 写到 U 盘 → U 盘变成"安装盘"
- 写到 UFS → UFS 变成"系统盘"

**你现在的做法是"写到 U 盘"，然后让里面的安装器再去写 UFS。** 这两步是分开的。

---

## 3. 先判断你的 img 属于哪一类

这一步决定后面走哪条路线，**必须先做**。

### 3.1 判断命令

**macOS / Linux：**
```bash
# 看文件类型
file system.img

# 看有没有分区表
fdisk -l system.img 2>/dev/null
# 或
diskutil list  # 需要先挂载
```

**Windows（PowerShell，需要 WSL 或 Git Bash）：**
```bash
# 用 7-Zip 打开看结构
7z l system.img
```

### 3.2 三类结果

| 结果 | 特征 | 类型 | 推荐路线 |
|---|---|---|---|
| **A. 有分区表的可启动安装器** | 启动后进入安装流程，自动装到内部盘 | **安装器镜像** | **路线 A：Ventoy** |
| **B. 纯系统盘镜像** | 就是 dd 出来的整盘系统，没有"安装器"概念 | **原始盘镜像** | **路线 B：dd** |
| **C. 混合** | 既可作为启动盘，也可直接写 | 看实际行为 | 先试路线 A |

### 3.3 怎么从行为上区分

| 观察点 | 安装器镜像（A） | 原始盘镜像（B） |
|---|---|---|
| 启动后 | 有进度条/安装界面/自动分区 | **直接进系统**或黑屏（缺驱动） |
| 分区动作 | 运行时会创建分区 | 分区在烧写时就已存在 |
| 大小 | 通常比系统实际占用大（含安装程序） | 等于整个系统盘大小 |

> 你的描述是"**从 U 盘启动后会格式化 UFS + 自动安装**"——这明显是**类型 A（安装器镜像）**。

---

## 4. 路线 A：Ventoy 直接启动 img

### 4.1 原理

Ventoy 的 **Disk 模式**会把 img 以"虚拟硬盘"的形式暴露给固件：

```
Rufus：img 逐字节写到 U 盘 → 固件看到一块"完整磁盘"
Ventoy：img 作为文件存放 → 启动时以虚拟块设备暴露 → 固件看到一块"完整磁盘"
```

**对安装器来说，两者没有本质区别。**

### 4.2 唯一的技术风险

安装器启动后，需要找到"自己"包含的：
- 安装脚本
- 系统镜像（WIM / raw / squashfs）
- 驱动包

**它找这些只有两种方式：**

| 寻找方式 | 换到 Ventoy 后 | 风险 |
|---|---|---|
| **按卷标 / 相对路径**（如 `INSTALL\`、卷标 `WINPE`） | ✅ 通常仍能找到 | 低 |
| **按固定设备路径**（`/dev/sdb1`、`\\.\PhysicalDrive1`） | ❌ 可能找不到或找错 | **高** |

> **这就是"Ventoy 能不能替代 Rufus"的唯一验证点。**

### 4.3 20 分钟验证方案

**不要一上来改造 20 台，先在一台机器上验证。**

```
① 找一根 U 盘，装 Ventoy（见 4.4）
② 把 img 拷到 /ISO/ 目录
③ 开机 → Ventoy 菜单 → 选中 img
④ 若提示选模式，优先选 "Disk" / "Normal mode"
⑤ 观察三个点：
   - ✅ 能启动到安装器界面
   - ✅ 安装器能找到源文件（不报"找不到安装源"）
   - ✅ 能识别并写入 UFS
```

**三种可能结果：**

| 结果 | 含义 | 下一步 |
|---|---|---|
| **完全正常** | 安装器不依赖固定设备号 | ✅ **直接换 Ventoy** |
| **能启动但找不到源** | 安装器按设备路径找 | 改 img 的查找逻辑，或改用路线 B |
| **根本启动不了** | 引导方式不兼容 | 保留 Rufus，或改走 dd |

### 4.4 安装 Ventoy

```bat
:: Windows 命令行（推荐）
Ventoy2Disk.exe VTOYCLI /I /Drive:F: /GPT /FS:NTFS
```

```bash
# Linux
sudo sh Ventoy2Disk.sh -i -g /dev/sdb
```

**推荐参数：**

| 参数 | 作用 |
|---|---|
| `/GPT` 或 `-g` | GPT 分区表（UEFI 机器） |
| `/FS:NTFS` | 镜像分区用 NTFS（大文件支持好） |
| `/R:8192` | 尾部预留 8GB（以后放数据分区） |

**结果判定**：生成 `cli_done.txt`，内容 `0` = 成功，`1` = 失败。

### 4.5 如果验证通过：Ventoy 方案的完整形态

```
【构建端】
1. img 不用改（已自带无人值守）
2. 上传服务器 / 放烧写站

【烧写端（只在第一次做）】
3. 装 Ventoy 到 U 盘
4. img 拷进 /ISO/
5. 配 ventoy.json 自动选中该 img（见 4.6）

【执行端（每台机器）】
6. 开机 → BIOS 第一启动项 = U 盘
7. Ventoy 倒计时 → 自动选中 img（Disk 模式）
8. 安装器自动格式化 UFS + 安装
9. 装完自动重启 → 需要切回内部盘（见第 6 章）
```

### 4.6 自动选中 img 的配置

`/ventoy/ventoy.json`：

```json
{
  "control": [
    { "VTOY_DEFAULT_IMAGE": "/ISO/system_v1.2.img" },
    { "VTOY_MENU_TIMEOUT": "3" },
    { "VTOY_DEFAULT_SEARCH_ROOT": "/ISO" }
  ]
}
```

| 参数 | 作用 | 注意 |
|---|---|---|
| `VTOY_DEFAULT_IMAGE` | 默认选中的镜像（`/` 开头全路径） | 改这一行就切换版本 |
| `VTOY_MENU_TIMEOUT` | 倒计时秒数，超时自动启动 | `3` 秒足够 |
| `VTOY_DEFAULT_SEARCH_ROOT` | 只搜索该目录 | 避免扫全盘 |

> ⚠️ **不要配 `auto_install` 插件**——它只对 Windows ISO 有效，对 img 无意义，配了反而可能干扰。

### 4.7 建议的 U 盘分区布局

```
┌─────────────────────────────────────────────────┐
│ 分区1：Ventoy 镜像区（Ventoy / NTFS，占主体）    │
│   /ISO/system_v1.2.img                          │
│   /ventoy/ventoy.json                           │
├─────────────────────────────────────────────────┤
│ 分区2：VTOYEFI（32MB，自动创建）                 │
├─────────────────────────────────────────────────┤
│ 分区3（可选）：数据区（exFAT）                   │
│   放多个版本的 img、日志、备用工具               │
└─────────────────────────────────────────────────┘
```

---

## 5. 路线 B：dd 整盘写入

### 5.1 什么时候用

| 情况 | 是否适用 |
|---|---|
| Ventoy 验证失败（安装器找不到源） | ✅ |
| img 是纯系统盘镜像（类型 B） | ✅ |
| 需要完全脚本化、不依赖交互 | ✅ |
| 需要写到内部 UFS 而不是 U 盘 | ✅ |

### 5.2 核心思想

**用 Ventoy 只负责"启动一个能跑 dd 的环境"，真正的写盘交给 dd。**

```
Ventoy → 启动 live linux / WinPE
        ↓
dd if=<img> of=<UFS设备> bs=16M
        ↓
sync → reboot
```

### 5.3 关键难点：img 从哪来

`dd` 需要同时：
- **读** img（源）
- **写** UFS（目标）

如果 img 和启动环境在**同一个 U 盘**，会冲突。三种解决方式：

#### 方式 1：U 盘双分区（离线，最稳）

```
分区1（启动区）：Ventoy + live iso
分区2（数据区）：/img/system_v1.2.img
```

启动后脚本挂载数据区，从那里读 img 写到 UFS。

#### 方式 2：网络拉取（推荐，无容量限制）

```bash
wget http://build-server/img/system_v1.2.img -O /tmp/disk.img
sha256sum -c /tmp/hash.txt
dd if=/tmp/disk.img of=/dev/sdX bs=16M status=progress conv=fsync
```

#### 方式 3：放到机器本地另一块盘

如果有第二块盘，img 可以预置在那里。

### 5.4 dd 命令与安全防护

**基础命令：**
```bash
sudo dd if=/path/to/system.img of=/dev/sdX bs=16M status=progress conv=fsync
```

**必须做的安全防护：**

```bash
#!/bin/bash
set -euo pipefail

IMG="$1"
EXPECTED_SIZE="${2:-}"   # 例如 "256G"，用于锚定目标盘

# ① 列出所有盘，人工/脚本确认
echo "=== 当前磁盘 ==="
lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN

# ② 用容量锚定目标盘（示例：找 256G 的盘）
if [ -n "$EXPECTED_SIZE" ]; then
    TARGET=$(lsblk -dno NAME,SIZE | awk -v s="$EXPECTED_SIZE" '$2==s {print $1}')
    [ -n "$TARGET" ] || { echo "未找到 $EXPECTED_SIZE 的盘"; exit 1; }
else
    read -rp "输入目标设备（如 sda）: " TARGET
fi

# ③ 确认不是 U 盘/系统盘
DEV_MODEL=$(lsblk -dno MODEL "/dev/$TARGET")
echo "目标：/dev/$TARGET ($DEV_MODEL)"
read -rp "确认写入？输入 YES: " ok
[ "$ok" = "YES" ] || exit 1

# ④ 写前卸挂载
umount "/dev/${TARGET}"* 2>/dev/null || true

# ⑤ 写入
echo "写入中..."
dd if="$IMG" of="/dev/$TARGET" bs=16M status=progress conv=fsync

# ⑥ 校验（可选但强烈建议）
IMG_BLOCKS=$(( $(stat -c%s "$IMG") / 16777216 ))
echo "校验中..."
dd if="/dev/$TARGET" bs=16M count="$IMG_BLOCKS" 2>/dev/null | sha256sum
sha256sum "$IMG"

echo "完成，重启..."
```

### 5.5 Windows 侧的 dd 替代

WinPE 里没有原生 dd，可选：

| 工具 | 说明 |
|---|---|
| **dd for Windows**（第三方 exe） | 单文件，可直接跑 |
| **Win32DiskImager 命令行** | 有 `/write` 参数 |
| **`diskpart` + 自定义写入器** | 只能分区，不能写 raw |
| **`dism`** | ❌ 只处理 WIM，不处理 raw img |

**WinPE 自动执行：** 通过 `startnet.cmd` 或 `winpeshl.ini`：

```bat
@echo off
wpeinit
:: 从数据区找 img
for %%d in (D E F G H I J) do (
    if exist %%d:\img\system.img (
        echo 找到镜像，开始写入...
        dd.exe if=%%d:\img\system.img of=\\.\PhysicalDrive0 bs=16M
        goto :done
    )
)
echo 未找到镜像
pause
exit
:done
wpeutil reboot
```

---

## 6. 通用难题：装完不要循环启动

无论路线 A 还是 B，**装完重启后如果 U 盘仍在、启动顺序仍是 U 盘优先，就会再次进入安装环境**。

### 6.1 三种解法

| 方法 | 原理 | 自动化 | 成本 | 可靠性 |
|---|---|---|---|---|
| **USB 继电器 / 可控 Hub** | 装完切断 U 盘数据线或供电 | **全自动** | $15~50/台 | 高 |
| **Ventoy 默认项改 Boot local disk** | 让 Ventoy 下次直接启动本地盘 | 半自动 | 0 | 中（需状态切换） |
| **人工拔盘** | 装完人拔 | 手动 | 0 | 高但费人 |

### 6.2 USB 继电器方案（推荐）

```
控制端：
  ① 检测到安装完成（安装器上报 / 看串口 / 看网络）
  ② POST /usb-relay/off  → 切断 U 盘
  ③ 发送重启
  ④ BIOS 启动项消失 → 自动 fallback 到内部盘
```

**实现选择：**

| 方式 | 说明 |
|---|---|
| 带独立开关的 USB Hub | 最便宜，但有型号兼容问题 |
| **USB 继电器模块**（串口/网络控制） | 最可靠，可编程 |
| **智能插座 + USB 供电分离** | 只能断供电（部分 U 盘断供电后仍会被枚举） |

### 6.3 Ventoy 配置切换方案

`VTOY_DEFAULT_IMAGE` 支持特殊值：

```json
{
  "control": [
    { "VTOY_DEFAULT_IMAGE": "F4>Boot the 1st local disk" },
    { "VTOY_MENU_TIMEOUT": "3" }
  ]
}
```

> ⚠️ **局限**：img 启动后是安装器在跑，**已经脱离 Ventoy**，没法回写 `ventoy.json`。
> 所以只有在"下一次重启前"由外部脚本改配置才行——本质还是需要一个控制端。

### 6.4 根治方案：BIOS 启动顺序 + 一次性启动

如果 BIOS 支持"一次性启动项"（One-time Boot Menu / Next Boot Device），可以：
- 每次需要装系统时，远程设置"下次从 USB 启动"
- 装完自动恢复默认（内部盘优先）

**这需要 BMC/KVM/固件管理接口**，普通消费级主板通常没有。

---

## 7. 网络路径：HTTP Boot / iPXE

如果你能摆脱 U 盘，这是最优雅的方案。

### 7.1 UEFI HTTP Boot

**前提**：固件支持（UEFI 2.5+ 且编入了 `HttpBootDxe`）。

```
固件 Boot URI = http://build-server/boot/bootx64.efi
        ↓
从 HTTP 拉引导文件
        ↓
进入 WinPE / live linux
        ↓
拉取 img → dd / 写入
```

**判断是否支持：**

```bash
# 进 UEFI Shell 后
http http://<server>/test.txt
```

能下载 → 固件有完整 HTTP 栈。

> ⚠️ **是 UEFI 固件能力，不是 Legacy BIOS 能力。**

### 7.2 iPXE 方案

```
固件 PXE / HTTP Boot → iPXE
        ↓
iPXE 脚本：下载 kernel + initrd + 执行自动脚本
        ↓
initrd 里 dd 写入
```

**优点**：不依赖固件的 HTTP Boot 支持（iPXE 自己实现完整协议栈）

### 7.3 三条网络路径对比

| 方案 | 依赖 | 复杂度 | 推荐度 |
|---|---|---|---|
| **UEFI HTTP Boot** | 固件支持 | 低 | ⭐⭐⭐（支持则首选） |
| **iPXE + HTTP** | 一段 PXE/HTTP 引导 | 中 | ⭐⭐ |
| **U 盘 + Ventoy** | 无 | 最低 | ⭐⭐⭐（通用） |

---

## 8. 自动化脚本

### 8.1 烧写站：同步 img 到 U 盘

```powershell
# sync-img.ps1
param(
    [Parameter(Mandatory)] [string]$ImgPath,
    [Parameter(Mandatory)] [string]$UsbDrive   # 例如 "F:"
)

$ErrorActionPreference = "Stop"
$imgName = Split-Path $ImgPath -Leaf
$destDir = Join-Path $UsbDrive "ISO"
New-Item -ItemType Directory -Force -Path $destDir | Out-Null

Write-Host "[1/3] 拷贝 $imgName ..."
Copy-Item $ImgPath (Join-Path $destDir $imgName) -Force

Write-Host "[2/3] 校验 SHA256 ..."
$src = (Get-FileHash $ImgPath -Algorithm SHA256).Hash
$dst = (Get-FileHash (Join-Path $destDir $imgName) -Algorithm SHA256).Hash
if ($src -ne $dst) { throw "校验失败：源 $src / 目标 $dst" }

Write-Host "[3/3] 更新 ventoy.json ..."
$ventoyDir = Join-Path $UsbDrive "ventoy"
New-Item -ItemType Directory -Force -Path $ventoyDir | Out-Null

@{
  control = @(
    @{ VTOY_DEFAULT_IMAGE = "/ISO/$imgName" },
    @{ VTOY_MENU_TIMEOUT = "3" },
    @{ VTOY_DEFAULT_SEARCH_ROOT = "/ISO" }
  )
} | ConvertTo-Json -Depth 5 | Set-Content (Join-Path $ventoyDir "ventoy.json") -Encoding UTF8

Write-Host "✅ 完成：$UsbDrive 已指向 $imgName"
```

### 8.2 Live Linux：自动烧写脚本

```bash
#!/bin/bash
# /opt/autoflash.sh —— 放在 live 系统的持久化分区里
set -euo pipefail

IMG_URL="http://build-server/img/system_v1.2.img"
IMG="/tmp/system.img"
EXPECTED_SIZE="${EXPECTED_SIZE:-256G}"

log() { echo "[$(date +%H:%M:%S)] $*" | tee -a /tmp/autoflash.log; }

log "开始自动烧写"

# ① 下载镜像
log "下载 $IMG_URL ..."
wget -q --show-progress -O "$IMG" "$IMG_URL"

# ② 校验
log "校验 SHA256 ..."
wget -q -O /tmp/hash.txt "${IMG_URL}.sha256"
( cd /tmp && sha256sum -c hash.txt ) || { log "校验失败"; exit 1; }

# ③ 定位目标盘
TARGET=$(lsblk -dno NAME,SIZE,TRAN | awk -v s="$EXPECTED_SIZE" '$2==s && $3!="usb" {print $1; exit}')
[ -n "$TARGET" ] || { log "未找到 $EXPECTED_SIZE 的目标盘"; exit 1; }
log "目标：/dev/$TARGET ($(lsblk -dno MODEL "/dev/$TARGET"))"

# ④ 卸载可能挂载的分区
umount "/dev/${TARGET}"* 2>/dev/null || true

# ⑤ 写入
log "写入中..."
dd if="$IMG" of="/dev/$TARGET" bs=16M status=progress conv=fsync
sync

# ⑥ 校验
BLOCKS=$(( $(stat -c%s "$IMG") / 16777216 ))
log "校验写入结果..."
dd if="/dev/$TARGET" bs=16M count="$BLOCKS" 2>/dev/null | sha256sum > /tmp/written.txt
log "源：$(sha256sum "$IMG" | awk '{print $1}')"
log "写：$(awk '{print $1}' /tmp/written.txt)"

# ⑦ 上报 + 重启
curl -s -X POST http://build-server:8080/api/flash-done \
     -H 'Content-Type: application/json' \
     -d "{\"host\":\"$(hostname)\",\"img\":\"$IMG_URL\"}" || true

log "完成，5 秒后重启"
sleep 5
reboot
```

### 8.3 安装完成上报（如果安装器支持）

如果 img 里的安装器支持调用外部命令，可以在装完后执行：

```bat
@echo off
powershell -NoProfile -Command ^
  "Invoke-RestMethod -Method Post -Uri 'http://build-server:8080/api/install-done' ^
   -Body (@{host=$env:COMPUTERNAME} | ConvertTo-Json) ^
   -ContentType 'application/json'"
```

### 8.4 控制端编排

```python
def reimage(host, img_version):
    # ① 准备 U 盘（只在版本变化时）
    if usb.version != img_version:
        sync_img_to_usb(img_version)
        usb.version = img_version

    # ② 触发重启（WOL / KVM / 智能插座）
    power_cycle(host)

    # ③ 等安装完成
    wait_for(lambda: api.get(f"/install-done/{host}") is not None,
             timeout=60 * 60)

    # ④ 断开 U 盘（继电器）或发出拔盘提示
    if usb_relay_available(host):
        usb_relay_off(host)

    # ⑤ 重启进新系统
    power_cycle(host)

    # ⑥ 验证新系统
    wait_for_ssh(host, timeout=15 * 60)
    assert get_os_version(host) == img_version
```

---

## 9. 性能预期

| 环节 | 耗时 | 瓶颈 |
|---|---|---|
| **Rufus 烧写（现状）** | 10~20 分钟/次 | USB 写速 |
| **Ventoy 拷 img（新方案）** | 1~5 分钟（随 img 大小） | USB 写速 |
| Ventoy 读取 img 启动 | 10~60 秒 | USB 读速 |
| 安装器格式化 + 安装 | **取决于安装器** | 磁盘写入 + CPU |
| **合计（Ventoy 路线）** | **比 Rufus 省掉 10~15 分钟** | — |

**不同 U 盘对"拷贝 img"的影响：**

| 类型 | 写速 | 20GB img 耗时 |
|---|---|---|
| USB 2.0 | ~20 MB/s | ~17 分钟 |
| USB 3.0 | ~100 MB/s | ~3.5 分钟 |
| USB 3.2 高速 | ~400 MB/s | ~50 秒 |

> **必须用 USB 3.0 以上**，否则"拷文件"的优势就没了。

---

## 10. 风险与排查

### 10.1 风险清单

| 风险 | 后果 | 应对 |
|---|---|---|
| **安装器按固定设备号找源** | Ventoy 下找不到安装源 | 先做 20 分钟验证（4.3），失败改 dd |
| **dd 写错设备** | 数据盘/U 盘全毁 | 用容量+型号+序列号三重锚定，加确认提示 |
| **装完循环启动** | 每次都重装，起不来 | USB 继电器 / 拔盘 / 改启动顺序 |
| **img 未做通用化（Windows）** | 换机器蓝屏/起不来 | 制作端先 `sysprep /generalize /oobe /shutdown` |
| **img 太大** | 拷贝/下载慢 | USB 3.0、分卷、或用网络流式写入 |
| **校验缺失** | 写坏了不知道 | 写前写后都算 SHA256 对比 |
| **U 盘寿命** | 频繁写入磨损 | Ventoy 方案写入量小（只拷一次） |
| **固件不认虚拟盘** | Ventoy Disk 模式启动失败 | 换 CD/DVD 模式试，或改 dd |

### 10.2 快速排查表

| 现象 | 可能原因 | 排查方向 |
|---|---|---|
| Ventoy 菜单没有 img | 文件没放 `/ISO/` 或扩展名不对 | 检查路径和文件名 |
| 选中 img 后黑屏 | 引导模式不兼容 | 试 Disk/CD 模式，试 UEFI/Legacy |
| 安装器报"找不到安装源" | 按设备号找源 | **改走 dd 路线** |
| 装机成功但重启又进安装 | 启动顺序仍指向 U 盘 | USB 继电器或拔盘 |
| dd 后起不来 | 目标盘未正确识别 / img 不完整 | 校验 SHA256、检查 `lsblk` |
| Windows img 蓝屏 | 未 sysprep | 制作端重新通用化 |

### 10.3 调试技巧

| 需求 | 做法 |
|---|---|
| 看安装器日志 | 装完进系统查日志目录；或接串口看输出 |
| 看 Ventoy 启动过程 | 启动时按 `F5` 切换调试模式 |
| 确认 img 内容 | `fdisk -l` / `7z l` / `file` |
| 确认目标盘 | `lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN` |
| 只读验证 | `mount -o loop,ro system.img /mnt` |

---

## 11. 待确认事项

以下信息将决定最终方案细节，请逐条确认：

### 11.1 img 的技术属性

- [ ] **文件大小**大约多少？（决定 U 盘选型和是否需要分卷）
- [ ] **有没有分区表？**（`fdisk -l` / `7z l` 的输出）
- [ ] **是 Windows 还是 Linux？**
- [ ] **如果是 Windows：做过 sysprep 吗？**
- [ ] img 里的**安装器是用什么写的？**（自研脚本？厂商工具？Ventoy？）

### 11.2 运行环境

- [ ] **UFS 在安装器里叫什么？**（`/dev/sda`、`/dev/mmcblk0`、`\\.\PhysicalDrive0`？）
- [ ] **目标机器有几块盘？**（是否只有一块 UFS？）
- [ ] **固件支持 HTTP Boot 吗？**（进 BIOS 找 `HTTP Boot` / `Boot URI`）

### 11.3 自动化要求

- [ ] **能接受"装完人拔 U 盘"吗？** 还是必须全自动？
- [ ] **单台机器重装频率？**（每天 / 每周）
- [ ] **是否已经有 IP-KVM / BMC？**

---

## 12. 建议的落地路径

| 阶段 | 时间 | 内容 | 产出 |
|---|---|---|---|
| **1. 判断 img 类型** | 10 分钟 | `fdisk -l` / `7z l` 看结构 | 确定走路线 A 还是 B |
| **2. 验证 Ventoy** | 20 分钟 | 一根 U 盘 + 一台机器实测 | 能否替代 Rufus |
| **3. 选定路线** | — | A 通过走 A；失败走 B | — |
| **4. 解决循环启动** | 1 天 | 先人拔盘验证，再上继电器 | 免人工 |
| **5. 脚本化** | 1~2 天 | 同步脚本 + 自动烧写脚本 | 一条命令完成 |
| **6. 批量铺开** | 按需 | 多根 U 盘 + 轮转流程 | — |

**核心原则**：**先花 20 分钟验证 Ventoy 能不能用，再决定投入。** 不要一上来就改 20 台。

---

## 参考链接

- Ventoy 官网：<https://www.ventoy.net/>
- Ventoy 使用说明：<https://www.ventoy.net/cn/doc_start.html>
- Ventoy Windows 命令行：<https://www.ventoy.net/cn/doc_windows_cli.html>
- Ventoy 全局控制插件：<https://www.ventoy.net/cn/plugin_control.html>
- Ventoy 磁盘布局说明：<https://www.ventoy.net/cn/doc_disk_layout.html>
- rufus 命令行能力（源码确认）：<https://github.com/pbatard/rufus/blob/master/src/rufus.c>

---

*文档随需求演进，img 的具体类型确认后需要相应调整第 4/5 章。*
