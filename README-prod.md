# Nano-V2 编译与烧录

环境：WSL2 Ubuntu、ESP-IDF v5.5.2；设备串口 `/dev/ttyACM0`。

## 编译、打包与烧录

在新终端中执行：

```bash
cd ~/github/xiaozhi-esp32
. ~/.espressif/tools/activate_idf_v5.5.2.sh
python scripts/release.py fogseek-nano-v2
idf.py -p /dev/ttyACM0 flash monitor
```

脚本自动重建 `sdkconfig`、选择板型、编译、合并和打包，无需手动执行 `set-target` 或修改 PATH。
**同版本同名 zip 已存在时会跳过构建，重新构建前请先移走旧包。**
脚本构建成功后再烧录，串口监控按 `Ctrl+]` 退出。普通烧录保留 Wi-Fi 和设备设置。

产物：

- 固件包：`releases/v{版本}_fogseek-nano-v2.zip`
- 合并固件：`build/merged-binary.bin`（独立烧录时写入地址 `0x0`）

## 配置

长期设置修改 `main/Kconfig.projbuild` 中 Nano-V2 对应的默认值，再重新构建、烧录。
release 会重建配置，只修改本地 `sdkconfig` 或 `menuconfig` 的值不会保留。
设备 NVS 如保存过 `wifi/ota_url`，该值优先于编译配置。

| 配置 | 当前值 |
|---|---|
| 板型 | `fogseek-nano-v2`，ESP32-S3 |
| OTA | `http://124.133.228.67:8002/xiaozhi/ota/` |
| Project-Id | `zhiban` |
| 配网 | BluFi 蓝牙配网，关闭热点配网 |
| 唤醒词 | 你好小智、嗨小巫 |
| 设备端 AEC | 未启用 |

芯片通用默认值在 `sdkconfig.defaults*`；板级 `config.json` 提供目标芯片与构建名称。

## WSL 串口挂载

若看不到 `/dev/ttyACM0`，在 Windows 管理员 PowerShell 执行：

```powershell
usbipd list
usbipd bind --busid <BUSID>        # 首次使用时执行
usbipd attach --wsl --busid <BUSID>
```

重插设备后需重新 attach。串口被占用时，先关闭其他监控程序。

## 常用操作

| 命令 | 用途 |
|---|---|
| `idf.py -p /dev/ttyACM0 flash monitor` | 烧录现有构建并查看日志 |
| `idf.py -p /dev/ttyACM0 monitor` | 仅查看日志 |
| `idf.py menuconfig` | 临时修改配置，随后执行 `idf.py build flash monitor` |
| `idf.py -p /dev/ttyACM0 erase-flash flash` | 擦除整片 Flash 后烧录，会清除 Wi-Fi 和其他设备设置 |

## 小程序蓝牙配网协议

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
