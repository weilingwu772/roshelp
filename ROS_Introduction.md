# ROS (Robot Operating System ) 入門

記錄ROS世界裡需先知道的觀念和關鍵字，後續要補的面項有：
- 打入機器人供應鏈，就必須接 ROS ，目前學校就是這樣教的
- 學生都很懶，SDK要上github，才會用那個硬體
- vibe-coding時要下的prompt

---

## 1. ROS演進對照表：從學習到產業應用

| 比較維度 | ROS 1 | ROS 2 |
| :--- | :--- | :--- |
| **定位** | 適合學習 | 商業開發 |
| **通訊核心** | 中心化 (依賴 roscore，一旦 master node 崩掉即通訊中斷) | 去中心化 (節點之間透過Discovery機制自動發現彼此，不再依賴 roscore 進行管理) |
| **主流演算法** | **Gmapping** (易學、適合新手) | **Cartographer** (精準、適合商用) |
| **硬體支援** | 電腦級 OS (Linux) | 跨平台與微控制器 (MCU) |
| **通訊品質** | 無 QoS | 支援 **QoS** |

---

## 2. ROS 2 的DDS技術底層
ROS 2 底層基於 **DDS (Data Distribution Service)** 工業標準，主要規範項目如下，業界通常使用免費開源版本，因此這並非零組件商或開發者(機器人整機商)會關注的議題，除非涉及極大圖資、極低延遲或多機大規模協作時，才會尋求專業公司提供優化方案。

### 2.1 建立資料可互通的機制
決定資料該長什麼樣子，確保跨語言（C++, Python, Java）的通訊是一致的。
- **IDL (Interface Definition Language)**：定義`message`（訊息）、`service`（服務）、`action`（動作）的標準格式語言。
- **XTypes (Extensible and Dynamic Topic Types)**：決定資料能不能互通的規則。
- **DCPS (Data-Centric Publish-Subscribe)**：定義 `DomainParticipant` (參加者)、`Topic` (主題)、`Publisher/Subscriber` (發布/訂閱者) 以及 `DataWriter/DataReader` 等物件的操作介面。

### 2.2 傳輸規則 QoS (Quality of Service)
預設 20 多種傳輸規則，開發者可自行設定或切換。
- **Reliability (可靠性)**：
    - `Reliable`: 確保資料必達（像傳簡訊，適合指令或重要狀態）。
    - `Best Effort`: 只求快，丟包不重傳（像看直播，適合感測器大數據）。
- **Durability (持久性)**：資料發布後，新加入的訂閱者是否能拿到之前的舊資料（例如地圖資料）。
- **Deadline (期限要求)**：資料多久必須更新一次。
- **History (歷史快取)**：系統要暫存多少筆數據在 Buffer 中。

### 2.3 安全規範 DDS Security 
規範工業/軍事等級的安全需求。
- **身分驗證 (Authentication)**：確認誰有資格加入通訊網。
- **存取控制 (Access Control)**：誰可以 Publish 哪一個 Topic。
- **加密 (Cryptography)**：資料在線路上的加密規則。

---
通訊協議 RTPS (Real-Time Publish-Subscribe)
確保不同廠商產品可以互通。
- **Wire Protocol**：規範資料在網路上流動的二進位格式。
- **自動發現 (Discovery)**：規範節點如何「自報家門」找到彼此，不需靠中心化的 Server。

---

## 3. 關鍵字與通訊協議

ROS 2 架構建立在「節點」通訊的基礎上。

### 3.1 節點 Nodes
節點是負責執行單一任務的程式（例如：一個讀取感測器數據，另一個控制馬達）。
- **特點**：一個機器人系統是由數十個甚至上百個 `node` 各司其職組成的。
- **去中心化**： ROS 1 會有一個主節點 master node，其他節點都要去跟主節點註冊，主節點會有一張表登記每個節點做的事情。 ROS 2 則沒有主節點管理機制，因此即使其中一個節點崩潰，其他功能仍能維持運作。

### 3.2 通訊機制 Communication
ROS 2 提供四種主要的通訊方式

```mermaid
%% ROS2 節點通訊機制示意圖
graph LR
    %% Topics 模式
    subgraph "Topics (非同步流)"
        Pub[節點: 發佈者] -->|訊息| T((話題 Topic))
        T -->|訊息| Sub[節點: 訂閱者]
    end

    %% Services 模式
    subgraph "Services (同步請求)"
        Cli[節點: 客戶端] -->|請求| S((服務 Service))
        S -->|回應| Ser[節點: 伺服端]
        Ser -.->|返回結果| Cli
    end

    %% Actions 模式
    subgraph "Actions (長時程任務)"
        ACli[節點: 行動客戶端] -->|目標| A((動作 Action))
        A -->|反饋/結果| ACli
        ASer[節點: 行動伺服端] <-->|執行與監控| A
    end
```

1.  **Topics (話題)**
    - **特點**：非同步、一對多或多對多。
    - **比喻**：就像廣播電台，感測器不停發送訊息，想聽的節點就自己訂閱。
    - **用途**：數據流（影像、雷達數據、馬達編碼器值）。
2.  **Services (服務)**
    - **特點**：請求/回應、同步等待。
    - **比喻**：就像去餐廳點餐，你發出請求後會在那邊等，直到對方給出結果（餐點）。
    - **用途**：開關燈、系統參數重設、單次的狀態查詢。
3.  **Actions (動作)**
    - **特點**：三段式通訊 (目標/回報/結果)、可取消、非同步。
    - **比喻**：就像叫人去執行一個長程專案，過程中會一直收到「進度 50%」的報告，你也可以中途叫他停下來。
    - **用途**：導航、機械臂執行路徑。
4.  **Parameters (參數)**
    - **用途**：儲存節點的靈魂（設定值），如最大速度、PID 參數等。

### 3.3 協定與介面 (Interface & Protocol) —— 溝通的「語言」與「規則」
**關鍵字：** `Interface`, `msg/srv/action`, `IDL`, `Serialization`, `CDR`

如果說「通訊機制」是寄件的模式 (Topic/Service)，那麼「協定與介面」就是**包裹內部的清單格式**。

> 💡 **白話比喻：**
> - **通訊機制** 是「打電話」或「傳信件」。
> - **協定介面** 是「我們都說繁體中文」以及「填報名表的欄位順序」。即使通訊線路通了，如果沒對齊介面（格式），雙方仍無法理解彼此。

1.  **介面格式 (Interfaces)**
    - **`.msg` (Message)**：定義 Topic 的內容（如：雷達數據包含角度、距離）。
    - **`.srv` (Service)**：定義「請求」與「回應」的雙向結構。
    - **`.action` (Action)**：定義「目標、進度反饋、最終結果」的三段式格式。
    - **目的**：實現**語言無關性**。不管是 C++ 或 Python 寫的節點，只要引用同一份 `.msg` 定義，就能百分之百確認資料結構一致。

2.  **序列化 (Serialization) —— 打包技術**
    - 系統如何把記憶體裡的變數轉成能透過網路傳輸的「二進位流 (Byte Stream)」。ROS 2 採用 **CDR (Common Data Representation)** 標準格式，確保資料在不同硬體架構（如 PC 與 MCU）間傳輸不會出錯。

3.  **與硬體協定的解耦 (Decoupling)**
    - **核心優勢**：ROS 將底層物理協定（如 Serial, CAN, USB）抽象化。開發者只需要關注 ROS 標準介面，不需要在演算法層次直接面對瑣碎的硬體通訊細節。

---

## 4. 開發環境與工程結構
**關鍵字：** `Workspace`, `Colcon`, `Package`, `Overlay/Underlay`

### 4.1 工作空間 (Workspace) —— 你的專屬實驗室
ROS2 的開發通常在 `colcon_ws` 中進行，資料夾結構有其嚴格意義：
- `src/`：**食材區 (源碼)**。放置所有的程式原始碼 (Packages)。
- `build/`：**烹飪過程 (中間檔)**。編譯時產生的臨時檔案，通常不需要手動修改。
- `install/`：**成品區 (執行檔)**。編譯成功後的產出。**重點**：編譯完後必須執行 `source install/setup.bash` 系統才找得到這些節點。

### 💡 核心觀念：Overlay (覆蓋層) 與 Underlay (底層)
- **Underlay**：指的是由作業系統安裝的 ROS2 全域環境（如 `/opt/ros/humble`）。
- **Overlay**：指你目前正在開發的 Workspace。
- **威力之處**：你可以開發一個與系統內建同名的 Package，只要 `source` 了 Overlay，系統就會「優先使用」你的版本。這就是 ROS 靈活微調的核心。

### 4.2 封裝 (Packages) —— 程式碼的最小組織
**關鍵字：** `Package Name`, `Dependency`, `Package 封裝`

每個 Package 是 ROS 2 程式碼的最小組織單位，確保功能的模組化與可攜性：
- **Package Name (套件識別字)**：每個套件的唯一身份符號，系統靠這個名字來定位（找檔案）並執行功能。
- **Package 封裝 (定義)**：將相關的原始碼、介面檔與配置檔打包在一起。
- **套件依賴 (Dependency)**：記錄在 `package.xml`，明確宣告「我要用到別人哪些工具（模組）」。
- **編譯手冊**：`CMakeLists.txt` (C++) 或 `setup.py` (Python) 紀錄了如何將原始碼轉化為可執行檔。

---

## 5. 系統觀測與操作 (CLI Tools) —— 你的 X 光機
熟練這些指令是為了具備「Introspection (自省)」能力，讓你看穿系統腦袋裡在想什麼：

### 5.1 核心運行與診斷
- `ros2 node list`：**查看點名表**。看看哪些節點（員工）正在運行。
- `ros2 topic list`：**查看看板清單**。確認有哪些通訊頻道在流動。
- `ros2 topic echo /topic_name`：**監視看板內容**。實時查看感測器數據或控制指令。
- `ros2 topic hz /topic_name`：**檢查頻率**。診斷系統是否有「延遲」或「掉幀」。

### 5.2 行動與設定
- `ros2 run package_name executable_name`：**手動派駐員工**。啟動單一節點。
- `ros2 service call /service_name service_type "{data: value}"`：**發送單次指令**。

### 💡 視覺化神器：`rqt_graph`
這是架構師最愛的工具。只需在終端機輸入 `rqt_graph`，系統會直接畫出所有節點與通訊的連線圖，讓你一眼看出「誰跟誰在講話」，是診斷斷線問題的最佳解。

---

## 6. 實戰範例：最小可行節點 (Minimal Node)
**關鍵字：** `rclpy`, `Node Class`, `Publisher`, `Timer Callback`, `Spin`

以下是一個簡單的 Talker 節點範例，展示 ROS2 的基本程式結構。

### 6.1 程式碼範例 (talker.py)
```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class SimpleTalker(Node): # 繼承 Node，代表招募一位「員工」
    def __init__(self):
        # 1. 初始化員工名稱
        super().__init__('simple_talker')
        
        # 2. 建立「看板 (Publisher)」：主題為 'chatter'，容量 10
        self.publisher_ = self.create_publisher(String, 'chatter', 10)
        
        # 3. 設定任務：每 0.5 秒執行一次 timer_callback
        # 💡 架構師提醒：不要在 ROS 使用 while True，會卡死通訊！
        self.timer = self.create_timer(0.5, self.timer_callback)
        self.i = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello ROS2: {self.i}'
        self.publisher_.publish(msg) # 將訊息貼到看板上
        self.get_logger().info(f'Publishing: "{msg.data}"')
        self.i += 1

def main(args=None):
    rclpy.init(args=args) # 開公司 (初始化環境)
    node = SimpleTalker()  # 派駐員工
    try:
        rclpy.spin(node)   # 進度迴圈：讓員工保持清醒，隨時處理任務
    except KeyboardInterrupt:
        pass
    node.destroy_node()    # 員工解僱
    rclpy.shutdown()       # 公司倒閉 (資源釋放)

if __name__ == '__main__':
    main()
```

### 6.2 重點解析
- **`super().__init__('name')`**：定義節點在 `ros2 node list` 中顯示的名字。
- **`create_timer`**：異步處理的關鍵。它讓節點在等待執行的空檔，還能去處理其他的 Service 請求或參數修改。
- **`rclpy.spin(node)`**：**這行最重要**。它是一個無限迴圈，負責監聽所有「事件」（定時器時間到、收到訊息等）。沒有它，節點執行完 main 就會直接結束退出。

---

## 7. 效能與維護：進階設計模式
**關鍵字：** `Composition`, `Lifecycle`, `Launch`, `Zero-copy`

### 7.1 Composition (組件化)
ROS2 允許將多個節點載入到同一個作業系統進程中，這稱為 **Composition**。
- **優點**：減少數據傳輸時的序列化開銷 (Zero-copy memory transport)，大幅提升性能。

### 7.2 Managed Nodes (Lifecycle Nodes)
生命週期節點允許對節點狀態進行精細控制（Unconfigured, Inactive, Active, Finalized）。
- **用途**：確保系統啟動時，所有感測器節點都已就緒，導航節點才開始運作。

### 7.3 Launch 檔案
使用 Python 或 XML 編寫 Launch 檔案，一次啟動數十個節點並設定它們的參數與命名空間。

---

## 8. 總結：知識網格建議
**關鍵字：** `CLI`, `Package`, `Custom Message`, `Network`

1.  **基礎**：學會使用 CLI 指令操作主題與服務。
2.  **開發**：嘗試用 Python 撰寫一個簡單的 Talker/Listener。
3.  **架構**：學習如何自定義 `.msg` 與 `.srv` 檔案。
4.  **進階**：深入研究 QoS 設定與多機器人組網。

---

## 9. 結論：從 Prototype 到 Product 的橋樑

隨著 ROS 1 在 2025 年正式停止維護，**ROS 2 已成為機器人開發的唯一標準**。它不只是版本的更新，更是從「實驗室雛型」向「工業級產品」的質變：

1.  **可靠性與安全性**：透過 DDS 技術，系統不再因單一節點（Master）崩饋而失效，並確保了數據傳輸的安全性。
2.  **更廣泛的適應性**：無論是微控制器（micro-ROS）還是高性能伺服器，ROS 2 都能提供穩定的開發範式。
3.  **生態系的成熟**：主流的 SLAM（如 Cartographer）、導航（Nav2）與感測器驅動均已在 ROS 2 上達到生產等級的效能。

**學習建議**：新進學習者應直接從 **ROS 2 Humble (LTS)** 或更新版本開始，以確保所學技術與未來產業趨勢接軌。

---
*文件更新於 2026-04-16*
