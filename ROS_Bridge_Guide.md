# ROS Bridge 圖解指南

ROSBridge 是一套讓「非 ROS 系統」與「ROS 世界」溝通的橋梁協定與工具套件。透過 WebSocket + JSON，任何語言、任何平台都可以與 ROS 節點互動，無需安裝 ROS 本身。

---

## 1. 核心概念：為何需要 Bridge？

ROS 內部使用 **DDS (Data Distribution Service)** 作為通訊中間件，這是一套高效但封閉的二進位協定，只有 ROS 節點才能直接溝通。

```mermaid
graph LR
    subgraph ROS2World ["ROS2 世界 (DDS)"]
        NA["Node A\n(C++/Python)"]
        NB["Node B\n(C++/Python)"]
        NA -->|DDS| NB
    end

    Browser["❌ 瀏覽器"]
    Mobile["❌ 手機 App"]
    Script["❌ Python 腳本\n(無完整 ROS 環境)"]

    Browser -. 無法直接加入 .-> ROS2World
    Mobile  -. 無法直接加入 .-> ROS2World
    Script  -. 無法直接加入 .-> ROS2World

    style ROS2World fill:#e8f5e9,stroke:#4CAF50
    style Browser fill:#ffebee,stroke:#f44336
    style Mobile fill:#ffebee,stroke:#f44336
    style Script fill:#ffebee,stroke:#f44336
```

**ROSBridge 的解法**：在 ROS 世界邊緣架一個「翻譯站」，將 DDS 訊息轉成通用的 JSON over WebSocket：

```mermaid
graph LR
    subgraph External ["外部世界"]
        B["瀏覽器 (JS)"]
        M["手機 App"]
        P["Python 腳本"]
    end

    subgraph Bridge ["翻譯站"]
        RS["rosbridge\nServer"]
    end

    subgraph ROS ["ROS 世界"]
        N["ROS 節點"]
    end

    B -->|WebSocket JSON| RS
    M -->|WebSocket JSON| RS
    P -->|WebSocket JSON| RS
    RS <-->|DDS| N

    style External fill:#e8f4f8,stroke:#2196F3
    style Bridge fill:#fff3e0,stroke:#FF9800
    style ROS fill:#e8f5e9,stroke:#4CAF50
```

---

## 2. 完整架構圖

```mermaid
graph LR
    subgraph External ["外部系統 (非 ROS)"]
        Browser["🌐 網頁瀏覽器\n(JavaScript / roslibjs)"]
        Mobile["📱 手機 App\n(iOS / Android)"]
        Script["🐍 Python 腳本\n(roslibpy)"]
        Unity["🎮 Unity / Unreal\n(模擬器 / 視覺化)"]
    end

    subgraph Bridge ["ROSBridge 翻譯站"]
        WS["WebSocket Server\n(Port 9090)"]
        Proto["rosbridge_protocol\n(JSON ↔ ROS Message)"]
    end

    subgraph ROS2 ["ROS2 環境 (DDS)"]
        CamNode["相機節點\n/camera/image_raw"]
        NavNode["導航節點\n/cmd_vel"]
        SrvNode["服務節點\n/set_mode"]
        TF["TF 座標轉換\n/tf /tf_static"]
    end

    Browser  -->|WebSocket JSON| WS
    Mobile   -->|WebSocket JSON| WS
    Script   -->|WebSocket JSON| WS
    Unity    -->|WebSocket JSON| WS

    WS <-->|內部轉換| Proto

    Proto <-->|DDS Topic| CamNode
    Proto <-->|DDS Topic| NavNode
    Proto <-->|DDS Service| SrvNode
    Proto <-->|DDS TF| TF

    style External fill:#e8f4f8,stroke:#2196F3
    style Bridge fill:#fff3e0,stroke:#FF9800
    style ROS2 fill:#e8f5e9,stroke:#4CAF50
```

---

## 3. ROSBridge 協定詳解

所有訊息皆為 **JSON 格式**，透過 WebSocket 傳輸。核心操作有三種：

### 3.1 Subscribe（訂閱 Topic）

外部系統向 ROSBridge 訂閱一個 Topic，當 ROS 節點發布訊息時，Bridge 自動推送給訂閱者。

```mermaid
sequenceDiagram
    participant Client as 外部用戶端<br/>(瀏覽器/App)
    participant Bridge as rosbridge_server
    participant Node as ROS 節點<br/>(/scan)

    Client->>Bridge: {"op": "subscribe",<br/>"topic": "/scan",<br/>"type": "sensor_msgs/LaserScan"}
    Note over Bridge: 向 ROS DDS 訂閱 /scan

    loop 每次感測器更新
        Node-->>Bridge: DDS 訊息 (二進位)
        Bridge-->>Client: {"op": "publish",<br/>"topic": "/scan",<br/>"msg": {...}}
    end

    Client->>Bridge: {"op": "unsubscribe",<br/>"topic": "/scan"}
```

### 3.2 Publish（發布 Topic）

外部系統透過 Bridge 向 ROS 發送指令，如控制機器人移動。

```mermaid
sequenceDiagram
    participant Client as 外部用戶端
    participant Bridge as rosbridge_server
    participant Node as ROS 節點<br/>(/cmd_vel)

    Client->>Bridge: {"op": "advertise",<br/>"topic": "/cmd_vel",<br/>"type": "geometry_msgs/Twist"}

    Client->>Bridge: {"op": "publish",<br/>"topic": "/cmd_vel",<br/>"msg": {"linear": {"x": 0.5},<br/>"angular": {"z": 0.1}}}

    Bridge->>Node: DDS 訊息 (Twist)
    Note over Node: 機器人開始移動 🤖
```

### 3.3 Service Call（呼叫服務）

外部系統呼叫 ROS Service，等待回應（類似 HTTP 請求）。

```mermaid
sequenceDiagram
    participant Client as 外部用戶端
    participant Bridge as rosbridge_server
    participant Srv as ROS 服務節點<br/>(/get_map)

    Client->>Bridge: {"op": "call_service",<br/>"service": "/get_map",<br/>"id": "req_001",<br/>"args": {}}

    Bridge->>Srv: DDS Service Request
    Srv-->>Bridge: DDS Service Response (地圖資料)

    Bridge-->>Client: {"op": "service_response",<br/>"service": "/get_map",<br/>"id": "req_001",<br/>"result": true,<br/>"values": {...}}
```

---

## 4. 典型部署架構

### 情境 A：機器人本機 + 遠端監控頁面

```mermaid
graph TD
    subgraph Robot ["機器人 (Raspberry Pi / Jetson)"]
        ROS2["ROS2 節點群"]
        Bridge["rosbridge_server\n:9090"]
        Web["靜態網頁伺服器\n:8080"]
    end

    subgraph RemotePC ["遠端電腦/手機"]
        UI["監控 Dashboard\n(roslibjs)"]
    end

    ROS2 <-->|DDS| Bridge
    UI   -->|HTTP| Web
    UI  <-->|WebSocket ws://robot-ip:9090| Bridge
```

### 情境 B：雲端控制台 + 多台機器人

```mermaid
graph TD
    Cloud["☁️ 雲端控制台\n(React / Vue)"]

    subgraph R1 ["機器人 1"]
        B1["rosbridge :9090"]
        N1["ROS2 節點"]
        B1 <--> N1
    end

    subgraph R2 ["機器人 2"]
        B2["rosbridge :9090"]
        N2["ROS2 節點"]
        B2 <--> N2
    end

    Cloud <-->|WebSocket| B1
    Cloud <-->|WebSocket| B2
```

---

## 5. 快速啟動

### 安裝

```bash
# ROS2 Humble
sudo apt install ros-humble-rosbridge-suite

# 或從 source
cd ~/ros2_ws/src
git clone https://github.com/RobotWebTools/rosbridge_suite
cd ~/ros2_ws && colcon build
```

### 啟動 Bridge Server

```bash
# 基本啟動（預設 Port 9090）
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# 指定 Port
ros2 launch rosbridge_server rosbridge_websocket_launch.xml port:=9091
```

### 前端 (JavaScript) 範例

```html
<!-- 引入 roslibjs -->
<script src="https://cdn.jsdelivr.net/npm/roslib/build/roslib.min.js"></script>

<script>
// 1. 建立連線
const ros = new ROSLIB.Ros({ url: 'ws://localhost:9090' });

ros.on('connection', () => console.log('✅ 已連線'));
ros.on('error',      (e) => console.log('❌ 錯誤', e));
ros.on('close',      () => console.log('🔌 已斷線'));

// 2. 訂閱感測器資料
const laserSub = new ROSLIB.Topic({
    ros,
    name: '/scan',
    messageType: 'sensor_msgs/LaserScan'
});
laserSub.subscribe((msg) => {
    console.log('雷射距離資料:', msg.ranges);
});

// 3. 發送移動指令
const cmdVel = new ROSLIB.Topic({
    ros,
    name: '/cmd_vel',
    messageType: 'geometry_msgs/Twist'
});
const twist = new ROSLIB.Message({
    linear:  { x: 0.2, y: 0, z: 0 },
    angular: { x: 0,   y: 0, z: 0.5 }
});
cmdVel.publish(twist);
</script>
```

### Python 範例 (roslibpy)

```python
import roslibpy

# 建立連線
client = roslibpy.Ros(host='localhost', port=9090)
client.run()

# 訂閱 Topic
listener = roslibpy.Topic(client, '/chatter', 'std_msgs/String')
listener.subscribe(lambda msg: print(f'收到: {msg["data"]}'))

# 發布 Topic
talker = roslibpy.Topic(client, '/cmd_vel', 'geometry_msgs/Twist')
talker.publish(roslibpy.Message({
    'linear':  {'x': 0.5, 'y': 0.0, 'z': 0.0},
    'angular': {'x': 0.0, 'y': 0.0, 'z': 0.0}
}))

client.termination_handler()
```

---

## 6. ROSBridge vs 其他方案比較

| 方案 | 協定 | 需要 ROS? | 適用情境 |
|------|------|-----------|----------|
| ROSBridge | WS + JSON | 否 | 網頁、跨平台 |
| ROS2 原生 DDS | DDS (RTPS) | 是 | ROS 節點互連 |
| micro-ROS | DDS (精簡版) | 部分 | 微控制器 |
| ROS1 Bridge | ROS1 ↔ ROS2 | 兩者都需要 | 系統遷移期 |
| MQTT + 自定義 | MQTT + JSON | 否 | IoT 設備 |

---

## 7. 注意事項

| 項目 | 說明 |
|------|------|
| **效能** | JSON 序列化比 DDS 二進位慢，不適合高頻率大量資料（如原始影像串流） |
| **安全** | 預設無驗證，生產環境應加 SSL (wss://) 與身份驗證 |
| **頻寬** | 訂閱高頻 Topic 前先評估網路負擔，可用 `throttle_rate` 限制推送頻率 |
| **替代品** | 影像串流建議改用 `web_video_server`；大量資料考慮 `foxglove-bridge` |

```mermaid
graph LR
    subgraph 適合用 ROSBridge
        A["監控 Dashboard"]
        B["遠端遙控介面"]
        C["資料視覺化"]
        D["跨語言腳本整合"]
    end

    subgraph 不建議用 ROSBridge
        E["原始影像串流\n→ 改用 web_video_server"]
        F["高頻 IMU 資料\n→ 改用 DDS 直連"]
        G["即時控制迴路\n→ 延遲過高"]
    end
```
