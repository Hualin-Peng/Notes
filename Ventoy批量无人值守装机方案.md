# Ventoy 批量无人值守装机方案

> 场景：多台 Windows 测试机，需频繁重装系统，**无法使用 PXE**。
> 原流程：Rufus 烧 img → 进 BIOS 选 U 盘启动 → 手动安装。
> 目标：用 Ventoy 把"每次重写 U 盘"变成"只拷一个文件"，实现半自动到全自动的装机流水线。
> 文档日期：2026-11-05

---

## 目录

- [1. 一句话原理](#1-一句话原理)
- [2. Ventoy 是怎么工作的](#2-ventoy-是怎么工作的)
- [3. 首次部署（做一次，之后不再碰）](#3-首次部署做一次之后不再碰)
- [4. 自动化核心：三个插件](#4-自动化核心三个插件)
- [5. 完整重装流程](#5-完整重装流程)
- [6. 最大难题：开机怎么自动进 Ventoy](#6-最大难题开机怎么自动进-ventoy)
- [7. 答案文件（unattend.xml）](#7-答案文件unattendxml)
- [8. 落地脚本](#8-落地脚本)
- [9. 性能预期](#9-性能预期)
- [10. 风险与排查](#10-风险与排查)
- [11. 与 Rufus 方案的对比](#11-与-rufus-方案的对比)
- [12. 落地步骤](#12-落地步骤)

---

## 1. 一句话原理

> **Ventoy 不"烧"镜像，它只是"启动"镜像。**
> 所以更新系统 = 往 U 盘拷一个新的 ISO 文件，几十秒完事，**永远不用再格式化 U 盘**。

这是整个方案的全部价值来源：

| | Rufus | **Ventoy** |
|---|---|---|
| 更新镜像 | 重写整盘（10~20 分钟） | **拷文件（30~90 秒）** |
| 一根盘装多个系统 | ❌ | ✅ 几十个 |
| 自动选默认镜像 | ❌ | ✅ 插件 |
| 无人值守安装 | ❌ 需重制 ISO | ✅ 原版 ISO + 答案文件 |
| 命令行可脚本化 | ❌ 只能预选，写盘必须点 GUI | ✅ 完整 CLI |

---

## 2. Ventoy 是怎么工作的

### 2.1 磁盘布局

安装后，U 盘被分成两个分区：

```
┌─────────────────────────────────────────────────────┐
│ 分区 1：镜像分区（卷标 Ventoy，exFAT/NTFS，占绝大部分）│
│   ├─ /ISO/Win11_24H2.iso                            │
│   ├─ /ventoy/ventoy.json          ← 插件配置        │
│   └─ /ventoy/script/unattend.xml  ← 答案文件        │
├─────────────────────────────────────────────────────┤
│ 分区 2：VTOYEFI（32MB，FAT，放引导文件）            │
├─────────────────────────────────────────────────────┤
│ 尾部（可选）：/R:2048 保留空间                       │
└─────────────────────────────────────────────────────┘
```

**关键理解：镜像分区就是一个普通文件系统。**

- 可以直接往里拖文件、删文件、建目录，跟普通 U 盘没区别
- 引导逻辑在 32MB 的 `VTOYEFI` 分区里，**不在镜像分区**
- 所以你把镜像分区重新格式化成 NTFS/FAT32 也不会破坏 Ventoy 引导
- 升级 Ventoy 只覆盖 `VTOYEFI`，**不会动你的 ISO**

### 2.2 它怎么"启动 ISO"

传统做法是"把 ISO 解压/写到盘上"，Ventoy 不是：

```
① 启动时扫描镜像分区
② 列出所有 .iso / .wim / .img / .vhdx / .efi 文件，按字母排序生成菜单
③ 选中某个 ISO 后，通过虚拟块设备（类似 loop 挂载）
   把 ISO 以只读块设备形式暴露给后续引导
④ 把控制权交给 ISO 里的引导程序（bootmgr / grub / syslinux）
```

**一个 U 盘能同时放几十个系统镜像，互不干扰**——这是 Rufus 完全做不到的。

### 2.3 为什么更新不用重烧

因为 **ISO 在 Ventoy 眼里只是一个普通文件**：

```
Rufus：更新 OS = 重新写整个 U 盘（覆盖分区表 + 全部数据）
Ventoy：更新 OS = 往 /ISO/ 里拷一个新 iso
```

---

## 3. 首次部署（做一次，之后不再碰）

### 3.1 安装方式

| 场景 | 命令 |
|---|---|
| Windows 图形界面 | `Ventoy2Disk.exe` → 选盘 → 安装 |
| **Windows 命令行**（推荐） | `Ventoy2Disk.exe VTOYCLI /I /Drive:F: /GPT /FS:NTFS` |
| Linux | `sudo sh Ventoy2Disk.sh -i -g -s /dev/sdb` |

**推荐参数组合：**

| 参数 | 作用 | 说明 |
|---|---|---|
| `/GPT` | 分区表用 GPT | UEFI 机器统一用 GPT |
| `/FS:NTFS` | 镜像分区文件系统 | 大容量盘用 NTFS（exFAT 对某些 UEFI 兼容性差） |
| `/NOSB` | 不启用 Secure Boot 支持 | 环境统一关闭 Secure Boot 时加 |
| `/R:2048` | 尾部保留 2048MB | 可选，留点空间 |
| `/PhyDrive:N` | 按物理盘号指定 | 替代 `/Drive:F:` |
| `/NOUSBCheck` | 跳过 USB 类型检查 | 用于内部硬盘，**谨慎** |

**Linux 参数对照：**

| Linux 选项 | 作用 |
|---|---|
| `-i` / `-I` | 安装 / 强制安装 |
| `-u` | 升级 |
| `-g` | GPT |
| `-s` | 启用 Secure Boot |
| `-r SIZE_MB` | 尾部保留空间 |
| `-L` | 镜像分区卷标 |

### 3.2 命令行安装的结果判定

Ventoy CLI 执行时会在**当前目录**生成三个文件：

| 文件 | 含义 |
|---|---|
| `cli_log.txt` | 完整日志 |
| `cli_percent.txt` | 进度百分比 |
| `cli_done.txt` | **完成标记，内容 `0` = 成功，`1` = 失败** |

脚本判断：

```powershell
$p = Start-Process -FilePath .\Ventoy2Disk.exe `
     -ArgumentList "VTOYCLI /I /Drive:F: /GPT /FS:NTFS /NOSB" `
     -Wait -PassThru -NoNewWindow
$done = Get-Content .\cli_done.txt -ErrorAction SilentlyContinue
if ($done -ne '0') { throw "Ventoy 安装失败，见 cli_log.txt" }
```

### 3.3 Secure Boot 说明

Ventoy 默认**开启 Secure Boot 支持**，方式是往 `VTOYEFI` 分区放自己的 MOK 证书，首次启动提示 enroll。

| 场景 | 做法 |
|---|---|
| 人工操作 | 按提示 `Enroll key from disk` → 应用（一次即可） |
| **批量运维（推荐）** | **统一关闭 Secure Boot**，或用 `/NOSB` 安装 |
| 必须开 Secure Boot | 提前把 Ventoy 证书预置进固件 |

⚠️ 如果不关 Secure Boot 也没 enroll，**每台机器第一次开机都会卡在 enroll 界面等人按**，自动化直接断。

### 3.4 关于安装在内部硬盘

Ventoy 也可以装到本地硬盘（`配置选项 → 显示所有设备`，或 Linux 直接指定设备）。

- **无损安装**可在已有数据的盘上做，但有严格前提：
  - 需要一个空闲分区表项（MBR 最多 4 个分区）
  - 第 1 个分区必须从 1MB 位置开始
  - 第 1 个分区只能是 NTFS（Linux 为 NTFS/EXT）
  - 第 1 个分区必须有剩余空间可压缩
- ⚠️ **会覆盖 MBR 及 MBR 到第 1 分区之间的 1MB 区域**，不建议在有启动的磁盘上做
- Windows 命令行带 `-n`：`sudo sh Ventoy2Disk.sh -i -n /dev/sdb`

---

## 4. 自动化核心：三个插件

所有插件配置都在镜像分区的 `/ventoy/ventoy.json`。

### 4.1 全局控制插件——决定"默认启动哪个镜像"

```json
{
  "control": [
    { "VTOY_DEFAULT_IMAGE": "/ISO/Win11_24H2.iso" },
    { "VTOY_MENU_TIMEOUT": "3" },
    { "VTOY_DEFAULT_SEARCH_ROOT": "/ISO" },
    { "VTOY_WIN11_BYPASS_CHECK": "1" },
    { "VTOY_SECURE_BOOT_POLICY": "0" }
  ]
}
```

| 参数 | 作用 | 对你的意义 |
|---|---|---|
| `VTOY_DEFAULT_IMAGE` | 默认选中哪个镜像（必须 `/` 开头的全路径） | **改这一行就切换要装的系统** |
| `VTOY_MENU_TIMEOUT` | 菜单倒计时（秒），超时自动启动默认项 | `3` 秒 → 基本无人干预 |
| `VTOY_DEFAULT_SEARCH_ROOT` | 只在指定目录下找镜像 | 避免扫全盘浪费时间 |
| `VTOY_WIN11_BYPASS_CHECK` | `1` = 绕过 Win11 TPM/内存/CPU 检查 | 测试机常缺 TPM，必开 |
| `VTOY_WIN11_BYPASS_NRO` | `1` = 绕过 Win11 在线账户要求 | 装本地账户用 |
| `VTOY_SECURE_BOOT_POLICY` | `0` = 不检查安全启动 | 关 Secure Boot 环境用 |
| `VTOY_DEFAULT_MENU_MODE` | `0` 列表 / `1` TreeView | 按喜好 |
| `VTOY_MENU_LANGUAGE` | 菜单语言，如 `zh_CN` | — |

**特殊情况**：`VTOY_DEFAULT_IMAGE` 还可设特殊值：

| 值 | 效果 |
|---|---|
| `VTOY_EXIT` | 超时后自动退出 Ventoy |
| `F4>Boot the 1st local disk` | 超时后直接启动本地第一块硬盘 |
| `F4>Search and boot Windows` | 超时后搜索并启动本机 Windows |

> 👉 **"Boot the 1st local disk" 是用来解决"装完循环进 Ventoy"的关键道具。**

**这就是"更新 OS"的全部秘密**：拷新 ISO → 改 `VTOY_DEFAULT_IMAGE` 指向它 → 完事。

### 4.2 自动安装插件——Windows 无人值守的关键

```json
{
  "auto_install": [
    {
      "image": "/ISO/Win11_24H2.iso",
      "template": "/ventoy/script/unattend.xml",
      "autosel": 1
    }
  ]
}
```

| 参数 | 说明 |
|---|---|
| `image` | 对应 ISO 的全路径，**支持模糊匹配**（如 `"/ISO/Win11*"`） |
| `parent` | 指定目录，该目录下所有 ISO 共用同一模板 |
| `template` | 答案文件路径，可以是字符串或数组（多套方案） |
| `autosel` | `1` = 默认选第 1 个模板；不设置时会弹确认菜单 |
| `timeout` | 弹菜单倒计时；`0` = 一直等；**不设置 + 设了 autosel = 直接自动选** |

**巨大优势：原版 Microsoft ISO 不需要重制。**

只要把 `unattend.xml` 放在 U 盘上，Ventoy 会在启动 ISO 时自动注入：

- ISO 的 SHA256 始终是原版，可校验、可追溯
- 换答案文件不用重做 ISO
- 一个 ISO 可配多套答案（开发机 / 测试机 / 性能机）

**支持的自动安装类型：**

| 系统 | 脚本格式 |
|---|---|
| Windows | Unattend XML |
| RHEL / CentOS / Fedora | Kickstart |
| Debian / 早期 Ubuntu Server | Preseed |
| Ubuntu Server 20.x+ | cloud-init |
| SUSE / openSUSE | autoYast |
| Deepin / UOS | INI |

### 4.3 变量扩展——避免格式化错盘（最救命的功能）

Ventoy 内置变量，可直接写进 `unattend.xml`，由 Ventoy 在启动时替换：

| 变量 | 含义 |
|---|---|
| `$$VT_WINDOWS_DISK_1ST_NONVTOY$$` | 第一个**非 Ventoy 盘**的磁盘号 |
| `$$VT_WINDOWS_DISK_1ST_NONUSB$$` | 第一个**非 USB 盘**的磁盘号 |
| `$$VT_WINDOWS_DISK_MAX_SIZE$$` | 容量最大的盘的磁盘号 |
| `$$VT_WINDOWS_DISK_NONVTOY_CLOSEST_300$$` | 容量最接近 300GB 的非 Ventoy 盘 |
| `$$VT_LINUX_DISK_SDX_1ST_NONVTOY$$` | Linux 下的第一个非 Ventoy 盘（sdX 形式） |
| `$$VT_LINUX_DISK_MAX_SIZE$$` | Linux 下容量最大的盘 |

**为什么重要**：硬编码 `Disk 0` 在不同机器上可能恰好是 U 盘，直接格式化错盘。用变量可以动态解析。

**约束**：
- 带变量的答案文件**必须是 UTF-8 编码**
- 每行最多 1 个变量
- 普通变量会在启动时弹框让你输入（不想弹框就只用 `$$VT_` 开头的内置变量）

---

## 5. 完整重装流程

```
① 构建服务器更新镜像
   \\build-server\os\Win11_24H2.iso

② 烧写站执行同步脚本（几十秒）
   - robocopy / Copy-Item 拷贝 ISO 到 U 盘 /ISO/
   - 自动生成 ventoy.json，指向新镜像
   - 拷贝 unattend.xml
   - 校验 SHA256

③ 把 U 盘接到 SUT
   - 人插：一次性
   - 常驻：每台机器插着不拔

④ 开机 / 远程重启
   - BIOS 第一启动项 = USB
   - Ventoy 倒计时 3 秒 → 自动启动 Win11_24H2.iso
   - Ventoy 自动注入 unattend.xml
   - Windows Setup 全自动：分区 → 展开 WIM → OOBE 配置
   - 首次登录执行 FirstLogonCommands / SetupComplete.cmd

⑤ 装完
   - SetupComplete.cmd 写入完成标记 / 上报控制端
   - 下次重启切回内部盘启动
```

---

## 6. 最大难题：开机怎么自动进 Ventoy

这是整个方案唯一卡脖子的点，**必须正面解决**。

### 6.1 方案 A：BIOS 启动顺序调一次（最常用）

进 BIOS 把 `UEFI: <Ventoy 盘名>` 设为第一启动项，机器上常驻插着 Ventoy U 盘。

- ✅ 之后每次重启自动进 Ventoy → 倒计时 → 自动装
- ❌ **装完必须让机器下次从内部盘启动，否则无限循环进 Ventoy**

### 6.2 解决"装完循环"的三种手段

| 手段 | 做法 | 自动化程度 | 成本 |
|---|---|---|---|
| **USB 继电器 / 可控 USB Hub** | 装完切断 U 盘数据线或供电，重启自然走内部盘 | **全自动** | 约 $15~50/台 |
| **Ventoy 侧配置切换** | 装完后让默认项指向 `F4>Boot the 1st local disk` | 半自动（需两阶段状态） | 0 |
| **人工拔盘** | 装完人拔 U 盘 | 手动 | 0 |

**方案：USB 继电器（推荐）**

```
控制端：POST /usb-relay/off  →  切断 U 盘数据
        →  发送重启
        →  BIOS 第一次启动项消失，自动 fallback 到内部盘
```

也可以用"带独立开关的 USB Hub"，串口或网络控制的型号都能做。

**Ventoy 侧配置切换的局限**：装完 ISO 启动后是 Windows Setup 在跑，此时**已经脱离 Ventoy**，没法回写 `ventoy.json`。所以只有在"下一次重启前"由外部脚本改配置才行——本质还是需要一个控制端，不如继电器干净。

### 6.3 方案 B：不用 USB，走内部盘 WinPE

如果每台机器内部都留一个 WinPE 分区（或已有 WinRE），**完全可以不要 U 盘**：

```
当前系统 → bcdedit 设一次性启动进 WinPE → 重启
WinPE 从网络拉 WIM → DISM /Apply-Image 写到内部盘 → bcdboot
→ 重启进新系统
```

- ✅ 没有介质、没有插拔、没有启动顺序问题
- ❌ 动的是本机 BCD，失败可能起不来 → 必须留救援路径（WinPE 分区常驻）

### 6.4 方案 C：UEFI HTTP Boot（如果固件支持，等于白送的 PXE）

现代 UEFI（2.5+）原生支持 **HTTP Boot**：在 BIOS 里把启动项设成 `http://server/bootx64.efi`，机器直接从 HTTP 拉引导文件。

- ✅ **不需要 DHCP 选项、不需要 PXE 广播**，绕开"PXE 不可用"
- ✅ 等于把"介质"彻底虚拟化，比 Ventoy 更彻底
- ⚠️ 取决于机器固件，**先查 BIOS 里有没有 `HTTP Boot` / `UEFI HTTP` 选项**

> 如果你的机器支持 HTTP Boot，**优先用它**——可以省掉整个 U 盘环节。

### 6.5 三方案选择建议

| 条件 | 推荐 |
|---|---|
| 能进 BIOS 改一次 + 愿意加 USB 继电器 | **方案 A（全自动）** |
| 机器内部有 WinRE / 能加 WinPE 分区 | 方案 B（最优雅，无介质） |
| 固件支持 HTTP Boot | **方案 C（首选）** |
| 只是偶尔重装 | 方案 A + 人工拔盘 |

---

## 7. 答案文件（unattend.xml）

### 7.1 完整骨架（Windows 11，UEFI + GPT）

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
  <!-- ========== Windows PE 阶段 ========== -->
  <settings pass="windowsPE">
    <component name="Microsoft-Windows-International-Core-WinPE"
               processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35"
               language="neutral" versionScope="nonSxS">
      <SetupUILanguage><UILanguage>zh-CN</UILanguage></SetupUILanguage>
      <InputLocale>0804:00000804</InputLocale>
      <SystemLocale>zh-CN</SystemLocale>
      <UILanguage>zh-CN</UILanguage>
      <UserLocale>zh-CN</UserLocale>
    </component>

    <component name="Microsoft-Windows-Setup"
               processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35"
               language="neutral" versionScope="nonSxS">
      <DiskConfiguration>
        <Disk wcm:action="add">
          <!-- 关键：动态解析非 Ventoy 盘，避免格式化错盘 -->
          <DiskID>$$VT_WINDOWS_DISK_1ST_NONVTOY$$</DiskID>
          <WillWipeDisk>true</WillWipeDisk>
          <CreatePartitions>
            <CreatePartition wcm:action="add">
              <Order>1</Order><Type>EFI</Type><Size>300</Size>
            </CreatePartition>
            <CreatePartition wcm:action="add">
              <Order>2</Order><Type>MSR</Type><Size>16</Size>
            </CreatePartition>
            <CreatePartition wcm:action="add">
              <Order>3</Order><Type>Primary</Type><Extend>true</Extend>
            </CreatePartition>
          </CreatePartitions>
          <ModifyPartitions>
            <ModifyPartition wcm:action="add">
              <Order>1</Order><PartitionID>1</PartitionID>
              <Format>FAT32</Format><Label>System</Label>
            </ModifyPartition>
            <ModifyPartition wcm:action="add">
              <Order>3</Order><PartitionID>3</PartitionID>
              <Format>NTFS</Format><Label>Windows</Label><Letter>C</Letter>
            </ModifyPartition>
          </ModifyPartitions>
        </Disk>
      </DiskConfiguration>

      <ImageInstall>
        <OSImage>
          <InstallFrom>
            <MetaData wcm:action="add">
              <Key>/IMAGE/INDEX</Key>
              <!-- 必须和 ISO 里 install.wim 的索引对上！ -->
              <Value>6</Value>
            </MetaData>
          </InstallFrom>
          <InstallTo>
            <DiskID>$$VT_WINDOWS_DISK_1ST_NONVTOY$$</DiskID>
            <PartitionID>3</PartitionID>
          </InstallTo>
        </OSImage>
      </ImageInstall>

      <UserData>
        <AcceptEula>true</AcceptEula>
        <ProductKey><Key>XXXXX-XXXXX-XXXXX-XXXXX-XXXXX</Key></ProductKey>
      </UserData>
    </component>
  </settings>

  <!-- ========== OOBE 阶段 ========== -->
  <settings pass="oobeSystem">
    <component name="Microsoft-Windows-Shell-Setup"
               processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35"
               language="neutral" versionScope="nonSxS">
      <OOBE>
        <HideEULAPage>true</HideEULAPage>
        <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
        <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
        <ProtectYourPC>3</ProtectYourPC>
      </OOBE>
      <UserAccounts>
        <LocalAccounts>
          <LocalAccount wcm:action="add">
            <Name>tester</Name>
            <Group>Administrators</Group>
            <Password>
              <Value>YourPassword</Value>
              <PlainText>true</PlainText>
            </Password>
          </LocalAccount>
        </LocalAccounts>
      </UserAccounts>
      <AutoLogon>
        <Enabled>true</Enabled>
        <Username>tester</Username>
        <Password><Value>YourPassword</Value><PlainText>true</PlainText></Password>
        <LogonCount>1</LogonCount>
      </AutoLogon>
      <FirstLogonCommands>
        <SynchronousCommand wcm:action="add">
          <Order>1</Order>
          <CommandLine>cmd /c echo done &gt; C:\reimage-flag.txt</CommandLine>
          <Description>写入完成标记</Description>
        </SynchronousCommand>
      </FirstLogonCommands>
    </component>
  </settings>
</unattend>
```

### 7.2 两个必踩的坑

**坑 1：`<Value>6</Value>` 的索引**

不同 ISO 的 `install.wim` 索引不一样。**不能猜**，先查：

```bat
dism /Get-WimInfo /WimFile:D:\sources\install.wim
```

专业版通常靠后，但以实际输出为准。

**坑 2：编码**

答案文件**必须是 UTF-8**。用 VS Code 另存为 `UTF-8`（**不要 `UTF-8 with BOM`**），否则可能注入失败或中文乱码。

### 7.3 装完自动执行的两种方式

| 方式 | 触发时机 | 位置 |
|---|---|---|
| `FirstLogonCommands` | 首次登录后，以用户身份 | 写在 unattend.xml 里 |
| `SetupComplete.cmd` | 安装完成、OOBE 之前，SYSTEM 身份 | ISO 的 `sources\$OEM$\$$\Setup\Scripts\` |

> `SetupComplete.cmd` 权限更高，适合装驱动、改注册表、加域。

### 7.4 如果需要加域

在 `oobeSystem` 的 `Microsoft-Windows-Shell-Setup` 里加：

```xml
<MachineObjectOU>OU=TestLab,DC=corp,DC=local</MachineObjectOU>
```

并在答案文件里提供域凭据（或用 `netdom join` 在 `SetupComplete.cmd` 里做，密码不落盘更安全）。

---

## 8. 落地脚本

### 8.1 同步镜像 + 校验（烧写站）

```powershell
# sync-image.ps1 —— 在烧写站执行
param(
    [Parameter(Mandatory)] [string]$IsoPath,
    [Parameter(Mandatory)] [string]$UsbDrive,   # 例如 "F:"
    [string]$Template = "\\build-server\os\unattend.xml"
)

$ErrorActionPreference = "Stop"
$isoName = Split-Path $IsoPath -Leaf
$destDir = Join-Path $UsbDrive "ISO"
New-Item -ItemType Directory -Force -Path $destDir | Out-Null

Write-Host "[1/4] 拷贝镜像 $isoName ..."
Copy-Item $IsoPath (Join-Path $destDir $isoName) -Force

Write-Host "[2/4] 校验 SHA256 ..."
$src = (Get-FileHash $IsoPath -Algorithm SHA256).Hash
$dst = (Get-FileHash (Join-Path $destDir $isoName) -Algorithm SHA256).Hash
if ($src -ne $dst) { throw "校验失败：源 $src / 目标 $dst" }

Write-Host "[3/4] 写入 ventoy.json（指向新镜像）..."
$ventoyDir = Join-Path $UsbDrive "ventoy"
$scriptDir = Join-Path $ventoyDir "script"
New-Item -ItemType Directory -Force -Path $scriptDir | Out-Null

@{
  control = @(
    @{ VTOY_DEFAULT_IMAGE = "/ISO/$isoName" },
    @{ VTOY_MENU_TIMEOUT = "3" },
    @{ VTOY_DEFAULT_SEARCH_ROOT = "/ISO" },
    @{ VTOY_WIN11_BYPASS_CHECK = "1" }
  )
  auto_install = @(
    @{ image = "/ISO/$isoName"; template = "/ventoy/script/unattend.xml"; autosel = 1 }
  )
} | ConvertTo-Json -Depth 6 | Set-Content (Join-Path $ventoyDir "ventoy.json") -Encoding UTF8

Write-Host "[4/4] 拷贝答案文件 ..."
Copy-Item $Template (Join-Path $scriptDir "unattend.xml") -Force

Write-Host "✅ 完成：$UsbDrive 已指向 $isoName"
```

### 8.2 批量初始化多根 U 盘

```powershell
# init-usbs.ps1 —— 并行初始化一批 U 盘
param([string[]]$Drives)

$Drives | ForEach-Object -Parallel {
    $d = $_
    Write-Host "初始化 $d ..."
    Start-Process -FilePath .\Ventoy2Disk.exe `
        -ArgumentList "VTOYCLI /I /Drive:$d /GPT /FS:NTFS /NOSB" `
        -Wait -NoNewWindow
    if ((Get-Content .\cli_done.txt -ErrorAction SilentlyContinue) -ne '0') {
        Write-Error "$d 安装失败"
    }
} -ThrottleLimit 4
```

> ⚠️ `cli_done.txt` 是共享文件名，并行执行会冲突。生产环境建议**串行初始化**，或每根盘用独立工作目录。

### 8.3 装完自动上报（SUT 端）

```bat
@echo off
REM SetupComplete.cmd —— 安装完成后以 SYSTEM 身份执行
powershell -NoProfile -Command ^
  "Invoke-RestMethod -Method Post -Uri 'http://build-server:8080/api/reimage-done' ^
   -Body (@{host=$env:COMPUTERNAME; ts=(Get-Date -Format o)} | ConvertTo-Json) ^
   -ContentType 'application/json'"
echo done > C:\reimage-flag.txt
```

### 8.4 控制端编排（伪代码）

```python
def reimage(host, iso_name):
    sync_image_to_usb(iso_name)              # 拷贝镜像 + 改 ventoy.json
    power_cycle(host)                        # WOL / 智能插座 / KVM
    wait_until_boot(host)                    # 等它从 U 盘启动
    wait_for(lambda: api.get(f"/flag/{host}") == "done",
             timeout=40 * 60)                # 等 SetupComplete 上报
    usb_relay_off(host)                      # 切断 U 盘
    power_cycle(host)                        # 重启进新系统
```

---

## 9. 性能预期

| 环节 | 耗时 | 瓶颈 |
|---|---|---|
| 拷贝 5GB ISO 到 U 盘 | 30~90 秒 | USB 3.0 写速 |
| Ventoy 读取 ISO 启动 | 10~30 秒 | USB 读速 |
| Windows Setup 展开 WIM | 5~15 分钟 | 磁盘写入 + CPU |
| 首次开机配置 | 3~10 分钟 | 机器性能 |
| **合计（无人值守）** | **约 15~30 分钟/台** | — |

**必须用 USB 3.0 盘。** USB 2.0 光读 5GB ISO 就要 2~3 分钟，整体可能翻倍。

**不同 U 盘速度参考：**

| 类型 | 顺序读 | 5GB ISO 读取时间 |
|---|---|---|
| USB 2.0 普通盘 | ~30 MB/s | ~3 分钟 |
| USB 3.0 普通盘 | ~120 MB/s | ~40 秒 |
| USB 3.2 高速盘 | ~400 MB/s | ~13 秒 |

---

## 10. 风险与排查

### 10.1 风险清单

| 风险 | 后果 | 应对 |
|---|---|---|
| **Secure Boot 未 enroll** | 卡在 MOK 界面 | 统一关 Secure Boot，或 `/NOSB` |
| **答案文件索引错** | 装成错误的 Windows 版本 | 部署前 `dism /Get-WimInfo` 核对 |
| **硬编码 Disk 0** | 格式化错盘（可能是 U 盘） | 用 `$$VT_WINDOWS_DISK_*$$` 变量 |
| **装完循环进 Ventoy** | 每次开机都重装 | USB 继电器 / 拔盘 / 改默认项 |
| **USB 2.0 盘** | 装一次一小时 | 只用 USB 3.0 |
| **答案文件编码** | 中文乱码 / 注入失败 | UTF-8 无 BOM |
| **U 盘寿命** | 频繁写入磨损 | Ventoy 方案写入量小，压力低 |
| **Windows 许可** | 法律风险 | 使用正版镜像与授权（VL/订阅） |
| **多盘符混淆** | 拷到了错误的 U 盘 | 脚本用卷标或序列号校验，别只用盘符 |

### 10.2 快速排查表

| 现象 | 可能原因 | 排查方向 |
|---|---|---|
| Ventoy 菜单不出现 | BIOS 未从 USB 启动 | 检查启动顺序、Secure Boot |
| 菜单出现但没自动选 | `VTOY_MENU_TIMEOUT` 未生效 | 检查 `ventoy.json` 路径和 JSON 语法 |
| 选了 ISO 但黑屏 | ISO 本身问题 / 引导模式不匹配 | 换 UEFI/Legacy 模式 |
| 安装卡在选分区 | 答案文件未注入 | 检查 `auto_install` 配置、路径大小写、UTF-8 |
| 装完重启又进安装 | 启动顺序仍指向 U 盘 | USB 继电器或拔盘 |
| `DiskID` 报错 | 变量未被替换 | 确认 Ventoy 版本 ≥ 1.0.77，文件名拼写 |
| 第二次启动找不到 ISO | 镜像分区被格式化/损坏 | 重新拷贝，检查文件系统 |

### 10.3 调试技巧

| 需求 | 做法 |
|---|---|
| 看 Ventoy 启动日志 | 启动时按 `F5` → 切换调试模式 |
| 看是否注入成功 | WinPE 里按 `Shift+F10` 开命令行，查 `X:\Windows\Panther\unattend.xml` |
| 看安装日志 | `C:\Windows\Panther\setupact.log` / `setuperr.log` |
| 手动测试镜像 | 不配 `auto_install`，看 ISO 本身能否启动 |

---

## 11. 与 Rufus 方案的对比

| | Rufus 手动 | **Ventoy 方案** |
|---|---|---|
| 更新镜像 | 重写整盘 10~20 分钟 | **拷文件 30~90 秒** |
| 一根盘多系统 | ❌ | ✅ |
| 自动选镜像 | ❌ | ✅ 插件 |
| 无人值守安装 | ❌ 需重制 ISO | ✅ 原版 ISO + 答案文件 |
| 命令行可脚本化 | ❌ 只能预选 | ✅ 完整 CLI |
| 动态磁盘号 | ❌ | ✅ 内置变量 |
| 是否需要 U 盘 | 要 | 要（可常驻） |
| 远程启动控制 | ❌ | ⚠️ 需 BIOS/继电器 |
| 成本 | 0 | **≈0（几根 U 盘）** |
| 适合规模 | 1~3 台 | **3~20 台** |

---

## 12. 落地步骤

| 阶段 | 时间 | 内容 | 验收点 |
|---|---|---|---|
| **1. 基础验证** | 半天 | 装 Ventoy 到 USB 3.0 盘，手动跑通"挂 ISO → 装完" | 能进 Ventoy 菜单并启动 ISO |
| **2. 无人值守** | 1~2 天 | 写 `unattend.xml`，验证磁盘变量和索引 | 全程零点击装完 |
| **3. 脚本化** | 1 天 | `sync-image.ps1` + `ventoy.json` 自动生成 | 一条命令换系统 |
| **4. 解决循环** | 1~2 天 | 先人拔盘验证，再决定是否上 USB 继电器 | 装完自动进新系统 |
| **5. 批量复制** | 1 天 | 初始化 N 根 U 盘，建立轮转流程 | 多台并行装 |

**核心原则：第 1~3 步零成本、纯收益，且产出的 ISO 仓库和答案文件在后续任何方案（IP-KVM、WinPE、HTTP Boot）里都 100% 复用。**

---

## 参考链接

- Ventoy 官网：<https://www.ventoy.net/>
- Ventoy 使用说明：<https://www.ventoy.net/cn/doc_start.html>
- Ventoy Windows 命令行：<https://www.ventoy.net/cn/doc_windows_cli.html>
- Ventoy 全局控制插件：<https://www.ventoy.net/cn/plugin_control.html>
- Ventoy 自动安装插件：<https://www.ventoy.net/cn/plugin_autoinstall.html>
- Ventoy 无损安装：<https://www.ventoy.net/cn/doc_non_destructive.html>
- Ventoy Secure Boot 说明：<https://www.ventoy.net/cn/doc_secure.html>
- Ventoy 启动配置替换插件：<https://www.ventoy.net/cn/plugin_bootconf_replace.html>

---

*价格、版本等信息随项目更新变化，部署前请以 Ventoy 官方文档为准。*
