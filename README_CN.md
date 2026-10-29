# esp32-c3-adblock

[English README](README.md) | [日本語 README](README_JP.md)

一款运行在 **2 美元 ESP32-C3** 上的 **Pi-hole 风格 DNS 广告拦截器**，*无需 PSRAM*。

> 📰 曾被 [Tom's Hardware](https://www.tomshardware.com/networking/clever-hacker-fits-537-000-domains-in-a-tiny-usd5-esp32-ad-blocking-dongle-firmware-uses-only-around-50kb-of-ram-and-can-answer-blocked-lookups-in-10-milliseconds)、[XDA Developers](https://www.xda-developers.com/this-tiny-esp32-powered-gadget-blocks-537000-domains-only-uses-50kb-of-ram/) 和 [Korben](https://korben.info/en/half-million-ad-blocking-domains-50kb-ram-esp32.html) 报道。

关键在于：你不必把整个拦截列表都放进 RAM。将域名存为**已排序的 40 位哈希值并写入闪存**，然后通过二分查找进行匹配。约 0.7 MB 闪存即可容纳 14 万多个域名，匹配耗时约 10 ms，RAM 使用量仅约 **50 KB**。

```text
收到查询 ──▶ 提取域名 ──▶ 计算 FNV-1a 哈希（也检查父级后缀）
         ──▶ 在闪存哈希表中进行二分查找
              ├─ 命中 ──▶ 返回 0.0.0.0（黑洞处理）
              └─ 未命中 ──▶ 转发到上游解析器，并中继响应
```

## 这一方案的特点

大多数 ESP32 DNS 黑洞会把拦截列表（域名*字符串*）加载到 RAM，因此需要 PSRAM。本项目改为将固定长度的 **5 字节（40 位）哈希值存储在闪存**中：

| | 字符串存入 RAM | 本项目（哈希存入闪存） |
| --- | --- | --- |
| 硬件 | ESP32 + PSRAM（约 8 美元） | ESP32-C3，无需 PSRAM（约 2 美元） |
| 141,000 个域名 | 约 2.5 MB RAM | **0.67 MB 闪存** |
| RAM 使用量 | 占用大部分 RAM | **约 50 KB** |
| 查找 | 字符串比较 | 约读取闪存 18 次（含 Wi-Fi 往返延迟约 10 ms） |
| 哈希冲突 | 不适用 | 141,000 个时为 0 次，537,000 个时为 1 次 |

**为什么选择 40 位？** 对当前闪存预算来说，这是最合适的折中。冲突概率遵循生日问题的规律：141,000 个域名时约为 0 次，537,000 个时约为 1 次（也就是说，可能会额外误拦截一个域名）。改用 32 位可以节省 20% 闪存，但 250,000 个域名时约会产生 7 次冲突；改用 64 位则会为一个并不存在的问题给每个域名多花 3 字节。

这个方法在更大的芯片上同样适用，并非 C3 的权宜之计。在 16 MB 闪存的 ESP32-S3 上，这些哈希值可容纳约 **270 万个域名**，而 8 MB PSRAM 存字符串只能容纳约 46.6 万个。基本上，在任何场景下，把哈希值存入闪存都比把字符串存入 PSRAM 更高效；C3 只是让这一点更加显而易见。

## 硬件

- 任意 **ESP32-C3** 开发板（已在 C3 SuperMini 上测试），4 MB 闪存，**无需 PSRAM**
- 经典 **ESP32**（DevKit / WROOM，4 MB）也可编译：`pio run -e esp32dev -t upload`（由社区贡献并通过编译测试；实际测试目标仍为 C3）
- 使用**稳定的 USB 电源**（例如手机充电器或路由器的 USB 接口）。廉价或连接松动的 USB-C 转 USB-A 转接头可能会在 Wi-Fi 发射时导致电压骤降。
- 使用 **USB-A 转 USB-C 转接头**，即可将设备直接插入大多数路由器背面的空闲 USB 接口，无需额外电源或设备外壳。

### 外壳

C3 SuperMini 的可打印外壳：[`hardware/esp32-c3-supermini-enclosure.stl`](hardware/esp32-c3-supermini-enclosure.stl)

打印注意事项：

- 无需支撑；层高 0.2 mm、填充率约 15% 即可。
- **保持天线一端通畅。** C3 的 PCB 天线是 USB-C 接口对侧短边上的 zigzag 走线。不要用实心塑料包住天线，也不要在附近放置金属，否则会影响 RSSI。
- 不要堵住通风口：开发板空闲时温度约为 45–55 °C。

## 构建和烧录（PlatformIO）

只需通过 USB 烧录一次，之后**固件和拦截列表都可以通过 Wi-Fi 更新**（见下文）。

> ⚠️ 请使用**当前版本的 PlatformIO**：可使用 VS Code PlatformIO 扩展自带的 Core，或在虚拟环境中运行 `pip install -U platformio`。发行版/apt 提供的 `platformio` 软件包（例如 4.3.4）版本太旧，会因 `AttributeError: ... 'resultcallback'` 而失败（issue #4）。一键浏览器安装程序正在开发中（托管位置待定）。

```bash
# 1. 复制密钥模板（该文件已加入 .gitignore，仅保留在本地）并进行编辑：
#    - WIFI_SSID / WIFI_PASS 为可选项——可以保留占位值，改用设备上的配置门户（见下文）。
#    - WEB_USER / WEB_PASS / OTA_PASS 为必填项：它们用于保护仪表板上会修改状态的接口
#      （/ban、/addblock、/upload、/update、/setupdate、/forgetwifi）以及网络 OTA。
#      请设置真实值；这些接口以前在局域网内任何人都可以访问。
cp src/secrets.example.h src/secrets.h
#    然后编辑 src/secrets.h

# 2. 构建拦截列表哈希表（默认使用 StevenBlack base + Hagezi Light，
#    约 10 万条，兼顾 WhatsApp 和社交网络的正常使用）
python3 tools/build_blocklist.py data/blocklist.bin

# 3. 烧录固件和拦截列表文件系统（唯一一次 USB 烧录）
pio run -t upload
pio run -t uploadfs

# 4. 查看启动信息、记录 IP 地址并打开仪表板
pio device monitor          # -> http://c3adblock.local
```

### 使用自己的拦截列表

`build_blocklist.py OUT.bin [SOURCE ...]` 可混合使用任意数量的 URL 和本地文件，支持以下格式：

- **hosts 文件** — `0.0.0.0 ads.example.com tracker.example.com`（该行中的所有域名都会被加入）
- **纯域名列表** — 每行一个域名
- **AdGuard / Adblock 基本规则** — `||ads.example.com^` 表示拦截，`@@||ok.example.com^` 表示移除域名（例如用于镜像 AdGuard Home 允许列表）

被拦截的域名，其子域名也会被拦截。DNS 哈希列表无法表达的规则（正则表达式、通配符、`$` 修饰符、隐藏元素规则 `##`）会被跳过并计数。`@@` 规则只会取消对完全匹配条目的拦截，无法从已拦截的父域名中单独放行某个子域名。如果某个来源无法下载，构建会停止，而不会静默生成较小的列表（可使用 `--allow-missing` 覆盖此行为）。

### Wi-Fi 配置（无需重新烧录）

如果设备无法连接 Wi-Fi（或从未设置 `secrets.h`），它会启动一个开放式接入点 **`C3-AdBlock-XXXX`** 和强制门户。用手机连接该接入点，选择网络并输入密码即可。以后要切换到新网络时，可在仪表板上点击 **Forget WiFi**，或在上电时按住 **BOOT** 按钮，重新打开配置门户。（现在访问 `/forgetwifi` 需要身份验证，不能再直接通过 URL 访问；详见下方“安全性”。）

## 无线更新（无需再用 USB）

**[http://c3adblock.local](http://c3adblock.local)** 上的仪表板可完成所有操作：

- **拦截列表** — 将刚生成的 `blocklist.bin` 上传到 *Blocklist → Upload*，或在 *Remote auto-update* 下设置 URL，让设备按计划拉取预先构建的 `blocklist.bin`。GitHub Actions 每周一都会重新构建默认列表，并发布到固定 URL。只需粘贴一次这个地址，设备就能自动保持更新：  
  `https://github.com/M-Abozaid/esp32-c3-adblock/releases/download/blocklist/blocklist.bin`
- **固件** — 在 *Firmware → OTA update* 中上传 `.pio/build/c3/firmware.bin`；设备会验证固件并重启到新映像。也可以通过命令行使用 Wi-Fi 推送：

  ```bash
  pio run -t upload --upload-port c3adblock.local --upload-protocol espota
  ```

**4 MB 闪存的取舍：**固件 OTA 需要两个应用分区，因此拦截列表可用空间约为 1.3 MB（**最多约 25 万个域名**）。激进的 53.7 万条 “ultimate” 列表只能放入单应用分区表（不支持固件 OTA）。请在 `partitions.csv` 中选择适合自己的方案。

## 安全性

仪表板的只读页面（`/`、`/stats.json`）仍可公开访问，但所有会修改状态的接口都需要 **HTTP Basic Auth**（在 `secrets.h` 中配置 `WEB_USER` / `WEB_PASS`）：

- `/ban`、`/addblock`、`/unblock`、`/forgetwifi`
- `/upload`、`/update`（拦截列表和固件 OTA）
- `/setupdate`、`/fetchnow`

网络 OTA（`ArduinoOTA`，例如运行 `pio run -t upload --upload-port c3adblock.local --upload-protocol espota`）需要同一文件中的 `OTA_PASS`。

如果没有这些保护，任何能访问设备所在局域网的人都可以刷入任意固件，或在没有任何凭据的情况下改写拦截列表。考虑到这台设备位于网络上所有 DNS 查询的路径中，这一点值得特别注意。此外，自定义拦截域名在显示到仪表板之前也会进行 HTML 转义，避免通过 `/addblock` 添加包含标记的域名后，在查看页面的浏览器中执行脚本，从而关闭存储型 XSS 漏洞。

**这里的 Basic Auth 是局域网信任边界控制，不是加密。**所有通信都是 :80 端口上的明文 HTTP——这款芯片没有足够的资源运行 TLS 服务器。Basic Auth 凭据采用 Base64 编码（并未加密），每次经过身份验证的请求都会发送；已经能够嗅探局域网流量的人（例如开放/访客 Wi-Fi 或 ARP 欺骗）可以从网络中读取凭据。这项措施用于防范常见情况——同一网络上的其他设备在没有凭据时访问 API，或浏览器标签页发起 CSRF 请求——而不是防范处于网络传输路径上的攻击者。

**缓存的 Basic Auth 凭据可能导致 CSRF：**浏览器会自动将缓存的 Basic Auth 凭据附加到已通过身份验证的来源上的*任何后续请求*，包括同一个浏览器之后访问完全无关网页时触发的请求（例如 `<img src="http://c3adblock.local/forgetwifi">`，不需要 JavaScript）。因此，只要曾经登录过仪表板，任何网页都可能静默调用此 API，与攻击者是否在同一局域网无关。现在，上述所有修改状态的接口还要求携带自定义 `X-Requested-With: c3-adblock` 请求头。普通的 `<img>` 或自动提交的 `<form>` 无法通过 CSRF 请求附加该请求头（只有同源 `fetch()` 可以，而仪表板自身的 JavaScript 正是这样调用接口的）。因此 `/forgetwifi` 不能再通过直接访问 URL 来调用，请使用仪表板上的按钮。

**默认凭据：**如果 `secrets.h` 仍使用 `secrets.example.h` 中的占位值 `CHANGE_ME_WEB_PASSWORD` / `CHANGE_ME_OTA_PASSWORD`，设备启动时使用的“密码”就是公开的（示例文件就在本仓库中）。出现这种情况时，固件会在串口日志中发出警告，并在仪表板显示提示横幅；但设备仍会正常启动和运行。因此，在将设备接入不完全可信的网络之前，务必先在 `secrets.h` 中设置真实密码。

目前尚未涵盖的情况：Wi-Fi 配置门户的接入点（`C3-AdBlock-XXXX`）仍会按设计以开放（未加密）方式运行——因为用户需要在尚未输入密码前就能连接。你在门户中输入的真实 Wi-Fi 密码，其安全性取决于配置期间这段短暂时间内的本地无线链路。

## 使用方法

将设备的 DNS 服务器设置为 C3 的 IP 地址，或将它添加为主 DNS 后面的**备用解析器**。可以用以下命令测试：

```bash
dig @<c3-ip> doubleclick.net   # -> 0.0.0.0（已拦截）
dig @<c3-ip> github.com        # -> 真实 IP（已转发）
```

## 注意事项（踩坑记录）

- **ModemManager**（Fedora/Ubuntu 默认安装）会占用 `/dev/ttyACM0` 并切换 DTR/RTS，这会**重置 C3**并阻断串口通信。解决方法：

  ```bash
  sudo systemctl stop ModemManager
  echo 'ATTRS{idVendor}=="303a", ENV{ID_MM_DEVICE_IGNORE}="1"' | sudo tee /etc/udev/rules.d/99-esp-no-modemmanager.rules
  sudo udevadm control --reload-rules && sudo udevadm trigger
  ```

- 在主机连接之前，C3 的 USB-Serial-JTAG 控制台可能会吞掉早期启动输出（`while(!Serial)` 有帮助）。
- DNS 客户端会添加 **EDNS OPT** 记录；被拦截查询的响应必须只包含问题和答案（`ANCOUNT=1`、`NSCOUNT=ARCOUNT=0`），否则报文格式无效。

## 已完成 / 后续计划

- ✅ Web 仪表板 — 按客户端统计拦截/放行次数、封禁客户端、添加自定义域名
- ✅ mDNS（`c3adblock.local`）设备发现
- ✅ OTA — 通过 Wi-Fi 更新固件和拦截列表，并按计划远程拉取拦截列表
- ✅ 强制门户 Wi-Fi 配置（无需硬编码凭据）+ 一键式浏览器 Web 安装程序
- ⬜ 分桶前缀索引 — 每次查找约读取闪存 18 次降至约 1–2 次（issue #3），提升吞吐量
- ⬜ 作为 DHCP 服务器（自动将自身设为 DNS），实现真正的即插即用

## 致谢

灵感来自 [s60sc/ESP32_AdBlocker](https://github.com/s60sc/ESP32_AdBlocker)——“对拦截列表中的域名返回 0.0.0.0”这一思路。本项目是独立的从头实现，重点针对无 PSRAM 芯片优化哈希存入闪存的方案。

## 许可证

MIT — 详见 [LICENSE](LICENSE)。
