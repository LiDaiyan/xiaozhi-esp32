# 编译与烧录说明（雾岸 nano-v2 / WSL2 环境）

> 本机环境：WSL2 (Ubuntu) + ESP-IDF v5.5.2（已装好）
> 目标板：雾岸科技 Nano-V2（ESP32-S3，16MB Flash + 8MB PSRAM）
> 串口：Windows USB → usbipd → WSL `/dev/ttyACM0`

---

## 一、环境准备（一次性，已配置好）

### 1. ESP-IDF 环境

IDF v5.5.2 已通过 EIM 安装在 `~/.espressif/v5.5.2/`。
**每个新开的终端都要先激活环境**：

```bash
. ~/.espressif/tools/activate_idf_v5.5.2.sh
idf.py --version   # 应显示 ESP-IDF v5.5.2
```

### 2. USB 串口（WSL2 需要 usbipd）

板子插在 Windows 的 USB 上，WSL2 默认看不到，需要挂载。
在 **Windows 管理员 PowerShell** 执行：

```powershell
usbipd list                        # 找到板子的 BUSID（VID:PID 303a:1001 那行）
usbipd bind --busid <BUSID>        # 只需做一次
usbipd attach --wsl --busid <BUSID>
```

挂载成功后 WSL 里 `ls /dev/ttyACM0` 应该存在。

⚠️ **重插板子 / 换 USB 口后，需要重新执行 `usbipd attach`**（busid 可能变，重新 `usbipd list` 查）。

---

## 二、日常开发：编译 + 烧录 + 看日志

```bash
cd ~/github/xiaozhi-esp32
. ~/.espressif/tools/activate_idf_v5.5.2.sh

idf.py build                          # 编译（增量，一般 1-2 分钟）
idf.py -p /dev/ttyACM0 flash          # 烧录
idf.py -p /dev/ttyACM0 erase-flash flash # 擦除整片 Flash 后重新烧录，包括清除 Wi-Fi、UUID 和其他保存的设置
idf.py -p /dev/ttyACM0 monitor        # 看串口日志（Ctrl+] 退出）
```

**一条龙写法**（编译+烧录+自动进入日志）：

```bash
idf.py -p /dev/ttyACM0 build flash monitor
```

### 当前 sdkconfig 已配置好的内容

| 配置项 | 当前值 |
|---|---|
| 目标芯片 | esp32s3 |
| 板型 | fogseek-nano-v2（`CONFIG_BOARD_TYPE_FOGSEEK_NANO_V2=y`） |
| OTA 地址 | `http://10.17.33.203:8002/xiaozhi/ota/` |
| 设备端回声消除 | 开（`CONFIG_USE_DEVICE_AEC=y`） |

> 直接 `idf.py build flash` 出来的固件即为本服务器可用版本，无需重新绑定（激活凭证和 WiFi 存在 NVS 分区，烧固件不影响）。

---

## 三、打包模式（release.py，产出给别人烧录的固件包）

```bash
. ~/.espressif/tools/activate_idf_v5.5.2.sh
cd ~/github/xiaozhi-esp32
python scripts/release.py fogseek-nano-v2
```

脚本会自动：选板型 → 按 `main/boards/fogseek-nano-v2/config.json` 附加配置 → 编译 → 合并 `build/merged-binary.bin` → 打包为 `releases/v{版本}_fogseek-nano-v2.zip`。

**拿到 zip 的人怎么烧**：
- Windows GUI：乐鑫 [Flash Download Tool](https://www.espressif.com/en/support/download/other-tools)，芯片选 ESP32-S3，把 zip 里的 `merged-binary.bin` 烧到地址 `0x0`
- 网页烧录（免安装）：https://espressif.github.io/esp-launchpad/

### ⚠️ release.py 的重要注意事项

1. **脚本内会执行 `idf.py set-target`，重新生成 `sdkconfig`**——手动改过的 OTA 地址会被抹掉（回退官方默认 `https://api.tenclass.net/xiaozhi/ota/`），板型则由脚本自动追加不受影响。
2. **解决办法**（让 OTA 地址永久生效）：在 `sdkconfig.defaults.esp32s3` 末尾加一行：
   ```
   CONFIG_OTA_URL="http://10.17.33.203:8002/xiaozhi/ota/"
   ```
   （该文件每次 set-target 都会生效；此文件在 git 里，不想提交就保持本地改动）
3. **release.py 只编译打包，不烧录**。想烧自己刚打的包：打包完直接 `idf.py -p /dev/ttyACM0 flash`（build 目录就是刚编译的产物）。

---

## 四、常用命令速查

| 命令 | 用途 | 注意 |
|---|---|---|
| `idf.py build` | 编译 | 增量编译 |
| `idf.py -p /dev/ttyACM0 flash` | 烧录 | 不擦 NVS，WiFi/绑定保留 |
| `idf.py -p /dev/ttyACM0 monitor` | 串口日志 | `Ctrl+]` 退出 |
| `idf.py -p /dev/ttyACM0 flash monitor` | 烧录+日志 | 最常用 |
| `idf.py menuconfig` | 图形改配置 | 板型：`Xiaozhi Assistant → Board Type`；OTA 地址也在里面 |
| `idf.py fullclean` | 清空 build 目录 | 换板型后或编译诡异报错时用，之后需重新 build |
| `idf.py erase-flash` | **全片擦除** | ⚠️ 清掉 WiFi 密码和激活绑定，之后要重新配网+输验证码绑定，慎用 |

---

## 五、常见问题

**Q：`idf.py: command not found`**
A：没激活环境，先 `. ~/.espressif/tools/activate_idf_v5.5.2.sh`。

**Q：烧录时报 `Could not open /dev/ttyACM0`**
A：串口被占用（别的 monitor 开着）或板子没挂载到 WSL。关掉其他串口程序；Windows 重新 `usbipd attach --wsl --busid <BUSID>`。

**Q：日志/烧录中途报 `device reports readiness to read but returned no data`**
A：WSL usbipd 断流（已知毛病）。Windows 端重新 attach，或重插板子。

**Q：换板子型号**
A：`idf.py menuconfig` → `Xiaozhi Assistant` → `Board Type` 里选；或直接改 `sdkconfig` 里的 `CONFIG_BOARD_TYPE_XXX=y`。改完建议 `idf.py fullclean` 再 build。

**Q：OTA 地址在哪改**
A：临时改：`sdkconfig` 文件里搜 `CONFIG_OTA_URL`；永久生效：写进 `sdkconfig.defaults.esp32s3`（见第三节）。改完重新编译烧录。

---

## 六、代码/配置位置速查

| 内容 | 位置 |
|---|---|
| 板级代码（引脚、外设初始化） | `main/boards/fogseek-nano-v2/fogseek-nano-v2.cc` |
| 板级引脚定义 | `main/boards/fogseek-nano-v2/config.h` |
| 板子编译附加配置 | `main/boards/fogseek-nano-v2/config.json` |
| 编译产物 | `build/xiaozhi.bin`、`build/merged-binary.bin` |
| 打包固件 | `releases/v{版本}_fogseek-nano-v2.zip` |
| 当前生效配置 | `sdkconfig`（生成文件，不入库） |
| 配置默认值 | `sdkconfig.defaults*`（入库） |

---

## 七、小程序蓝牙配网协议

小程序与音箱通过 BluFi 协议通信。下文的 JSON 是 BluFi 数据帧的业务载荷，
不能直接把普通 JSON 字符串写入蓝牙特征值；客户端需要处理 BluFi 帧格式、分包和校验。

| 项目 | UUID |
|---|---|
| BluFi 服务 | `0000FFFF-0000-1000-8000-00805F9B34FB` |
| 小程序写入特征 | `0000FF01-0000-1000-8000-00805F9B34FB` |
| 音箱通知特征 | `0000FF02-0000-1000-8000-00805F9B34FB` |

### 1. 蓝牙连接后查询设备信息

小程序完成蓝牙连接、开启通知后，主动发送 BluFi Custom Data（子类型 `0x13`）：

```json
{"cmd":"get_device_info"}
```

音箱收到请求后，通过同一 Custom Data 类型返回：

```json
{
  "type": "device_info",
  "deviceid": "9c:13:9e:e5:97:b0",
  "name": "Xiaozhi-Blufi"
}
```

- `type`：响应类型，固定为 `device_info`。
- `deviceid`：设备 Wi-Fi STA MAC；示例值属于当前测试板，每块设备不同。
  它与设备请求 OTA 和建立 WebSocket 连接时的 `Device-Id` Header 一致。
- `name`：设备蓝牙名称，当前为 `Xiaozhi-Blufi`。

音箱不会仅因蓝牙连接成功就自动发送这段 JSON，需要小程序先发送查询请求。
这里的 `deviceid` 不是 UUID／`Client-Id`，也不是微信蓝牙 API 返回的 `deviceId`。
实现位置：`main/boards/common/blufi.cpp` 的 `ESP_BLUFI_EVENT_RECV_CUSTOM_DATA` 分支。

### 2. Wi-Fi 连接结果通知

小程序通过标准 BluFi 消息发送 SSID、密码和连接请求后，音箱尝试连接路由器，
并使用 `esp_blufi_send_wifi_conn_report()` 返回 Wi-Fi Connection Report（子类型 `0x0F`）。

这条通知是二进制消息，不是 JSON，其 BluFi 载荷结构为：

```text
[Wi-Fi 模式: 1 字节]
[连接状态: 1 字节]
[SoftAP 连接数: 1 字节]
[可选附加字段: 类型、长度、内容 …]
```

当前 STA 模式下，连接成功通知的基本载荷为 `01 00 00`，随后附加 SSID 和路由器 BSSID。
小程序当前读取前两个字节，解析后的业务对象相当于：

```json
{
  "mode": 1,
  "state": 0
}
```

`mode=1` 表示 STA 模式。`state` 的含义为：

| 值 | 含义 |
|---|---|
| `0` | 连接成功并获得 IP |
| `1` | 连接失败 |
| `2` | 正在连接 |
| `3` | 已连接路由器，但尚未获得 IP |

**连接结果通知不包含设备 `deviceid`。** 附加字段中的 BSSID 是路由器的 MAC，
不能用它代替设备的 Device-Id。

### 3. 小程序对接顺序

1. 连接蓝牙，开启通知。
2. 发送 `get_device_info`，保存音箱返回的 `deviceid`。
3. 发送 Wi-Fi SSID、密码和连接请求。
4. 收到 `state=0` 的连接结果后，使用先前保存的 `deviceid` 向业务服务端申请用户绑定。
   固件本身不处理用户账号绑定，绑定接口由小程序和服务端另行实现。
5. 小程序收到成功结果后主动关闭蓝牙连接，音箱在断开回调中释放蓝牙资源。

以上是蓝牙消息交互，与设备联网后向 OTA 服务端发起的 HTTP 请求属于不同流程。
