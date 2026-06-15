# PairDrop 專案架構與核心機制分析報告

PairDrop 是一個基於瀏覽器的跨平台本地及網路檔案傳輸工具，啟發自 Apple 的 AirDrop。它是知名開源專案 [Snapdrop](https://github.com/RobinLinus/snapdrop) 的分支（Fork），並在原版的基礎上進行了大幅的擴展與穩定性改良。

本報告將從**核心架構**、**後端模組**、**前端模組**、**核心運作流程**，以及**與 Snapdrop 的差異**五個面向進行深入分析。

---

## 1. 核心技術棧與架構設計

PairDrop 採用了**「輕量前端 PWA + 後端 Node.js 信令伺服器」**的去中心化 P2P 架構：

*   **前端（Frontend）**：使用 Vanilla HTML5 / ES6 JavaScript / CSS3。沒有採用如 React 或 Vue 等重型框架，而是自行實現了基於事件發佈/訂閱（Pub/Sub）的 UI 驅動機制。前端核心功能依靠 WebRTC（Web Real-Time Communication）實現端到端的 P2P 資料傳輸。
*   **後端（Backend）**：純 Node.js 實現（Express + `ws` WebSocket 庫）。摒棄了 Snapdrop 原本的 Python 依賴，使專案更容易在樹莓派（Raspberry Pi）或 Docker 容器中部署。後端不儲存或傳輸用戶的實際檔案，主要職責是**靜態資源託管**、**WebRTC 信令交換（Signaling）**以及**房間與連線管理**。
*   **數據流向**：
    ```mermaid
    sequenceDiagram
        autonumber
        actor PeerA as 用戶 A (瀏覽器)
        participant WS as Node.js 信令伺服器
        actor PeerB as 用戶 B (瀏覽器)

        Note over PeerA, PeerB: 發現與信令階段
        PeerA->>WS: WebSocket 連線 (傳送 IP & PeerID)
        PeerB->>WS: WebSocket 連線 (傳送 IP & PeerID)
        WS-->>PeerA: 廣播房間內的 Peers 列表 (包含 PeerB)
        WS-->>PeerB: 廣播房間內的 Peers 列表 (包含 PeerA)

        Note over PeerA, PeerB: WebRTC 握手階段 (透過 WS 轉發)
        PeerA->>WS: 發送 WebRTC Offer (Signal)
        WS->>PeerB: 轉發 WebRTC Offer
        PeerB->>WS: 發送 WebRTC Answer (Signal)
        WS->>PeerA: 轉發 WebRTC Answer
        PeerA->>WS: 交換 ICE Candidates
        WS->>PeerB: 轉發 ICE Candidates

        Note over PeerA, PeerB: 檔案傳輸階段 (P2P 直連)
        PeerA->>PeerB: 建立 WebRTC Data Channel
        PeerA->>PeerB: 直接傳輸檔案分片 (不經過伺服器)
    ```

---

## 2. 後端模組分析 (Server-side)

後端代碼結構清晰，主要位於 `server/` 目錄下：

| 檔案名稱 | 職責與核心邏輯 |
| :--- | :--- |
| **`server/index.js`** | **系統入口與環境變數解析**。<br>1. 解析連接埠、Debug 模式、限流設定（`RateLimit`）、自定義 STUN/TURN 設定（`RTC_CONFIG`）、中轉備用方案（`WS_FALLBACK`）等參數。<br>2. 處理異常捕獲（如 `uncaughtException`）以實現自動重啟。<br>3. 啟動 `PairDropServer` 與 `PairDropWsServer`。 |
| **`server/server.js`** | **Express HTTP 伺服器**。<br>1. 託管 `public/` 下的靜態資源（PWA 用戶端網頁）。<br>2. 設定基於 `express-rate-limit` 的 API 訪問速率控制，保護伺服器免受惡意攻擊。<br>3. 提供 `/config` 路由，讓前端能動態獲取當前實例的 STUN/TURN 伺服器與按鈕設定。<br>4. 提供 `/ip` 路由用於調試反向代理下的真實客戶端 IP。 |
| **`server/ws-server.js`** | **WebSocket 信令與房間管理器 (核心)**。<br>1. 處理 WebSocket 連線與保活機制（Ping-Pong）。<br>2. 管理三種「房間」型態：區域網路房間 (`ip`）、配對房間（`secret`）、臨時公共房間（`public-id`）。<br>3. 負責轉發 WebRTC 信令（SDP 與 ICE Candidate）。<br>4. 當 `WS_FALLBACK` 開啟且 WebRTC 連線失敗時，此模組會直接在 WS 通道上轉發檔案分片（中轉模式）。 |
| **`server/peer.js`** | **Peer（客戶端）狀態封裝**。<br>1. 取得並規範化客戶端的真實 IP。若為私有 IP（LAN），則統一標記為 `127.0.0.1` 房，確保同區域網路用戶自動歸於同房。<br>2. 使用 `ua-parser-js` 解析 User-Agent，提取 OS、瀏覽器、設備類型等資訊以決定前端顯示的圖示。<br>3. 基於 `unique-names-generator` 為每個設備生成隨機名稱（例如：*Golden Salmon*），該名稱基於 PeerID 雜湊，確保同一標籤頁重新載入後名稱不變。 |
| **`server/helper.js`** | **工具函式庫**。實現包括高性能哈希（`cyrb53`，用於設備命名隨機數種子）以及加密安全的隨機字串生成器（`randomizer`，用於生成 256 位元的 Room Secret）。 |

---

## 3. 前端模組分析 (Client-side)

前端代碼主要位於 `public/` 目錄，核心邏輯在 `public/scripts/` 中。PairDrop 的前端採用了**延遲加載（Deferred Loading）**技術，先加載基本骨架，渲染背景與加載動畫，隨後異步下載重型腳本（如 WebRTC 包、ZIP 壓縮庫等），大大提升了 PWA 的首屏載入速度。

### 3.1 核心前端架構組件
*   **`main.js`**：整個前端的 Bootstrap。註冊 Service Worker（支援離線運行與 PWA 安裝），初始化各個 UI 控制器，並在 WebSocket 連線建立後評估 URL 參數（例如：處理配對連結或加入房間的邀請）。
*   **`network.js` (關鍵網路層)**：
    *   `ServerConnection`：管理與 WebSocket 信令伺服器的長連線，處理重連、保活與伺服器回傳事件。
    *   `Peer` (基底類別)：定義了檔案傳輸的通用協議（如：請求、回應、頭部傳送、分片控制、進度通知、完成確認）。
    *   `RTCPeer` (繼承自 `Peer`)：WebRTC P2P 傳輸的具體實現。管理 `RTCPeerConnection`、SDP 握手、ICE 收集，並開啟 `RTCDataChannel` 以二進位流形式直接收發數據。
    *   `WSPeer` (繼承自 `Peer`)：當雙方無法建立 WebRTC P2P 通道時，透過 WebSocket 伺服器中轉分片數據的備用通道。
    *   `PeersManager`：協調管理當前所有已知 Peer（無論是同區域網路、已配對還是公共房間內的設備），將底層網路事件轉化為全域 UI 事件。
    *   `FileChunker` / `FileDigester`：
        *   `FileChunker`：將大檔案切割為固定大小（64 KB）的分片，並以 1 MB 為單位分批（Partition）送出，每批傳送完畢會等待接收方的 `partition-received` 確認，**避免發送過快導致瀏覽器緩衝區溢出**。
        *   `FileDigester`：接收端將零散的 binary chunks 重新拼接，並實時計算傳輸進度，傳輸完畢後組裝成完整的 `Blob` 並執行下載。

*   **`ui.js` & `ui-main.js` (UI 展現層)**：
    *   `PeersUI` 與 `PeerUI`：管理主畫面上代表各個設備的「圓圈」格點。動態呈現設備圖示、連線類型標記、傳輸時的環形進度條。
    *   `Dialog` 繼承體系：包含多個模態視窗，例如持久配對 (`PairDeviceDialog`)、設備編輯 (`EditPairedDevicesDialog`)、公共房間 (`PublicRoomDialog`)、文字傳送與接收 (`SendTextDialog` / `ReceiveTextDialog`) 等。
    *   `ThemeUI`：切換 Light / Dark 模式，或跟隨系統主題。
    *   `BackgroundCanvas`：繪製 PairDrop 標誌性的動態水波紋背景。
*   **`persistent-storage.js`**：封裝 IndexedDB 與 LocalStorage，用於在瀏覽器端持久化儲存配對設備的 Room Secrets、自訂名稱、自動接受傳輸（Auto-Accept）設定等資訊。
*   **`localization.js`**：處理多語系。透過 `fetch` 加載對應的 `lang/*.json`（支援繁體中文 `zh-TW`、簡體中文 `zh-CN` 等多達 30 餘種語言），動態翻譯帶有 `data-i18n-key` 屬性的 DOM 元素。
*   **`browser-tabs-connector.js`**：解決「同一個瀏覽器開啟多個 PairDrop 標籤頁」的痛點。它使用 LocalStorage 與 BroadcastChannel 同步狀態，防止同設備的標籤頁之間發生混亂，並避免重複連線。

---

## 4. 核心運作流程與房間機制

PairDrop 最具特色的部分是其豐富的**設備發現與連線房間**機制：

### 4.1 房間類型
1.  **IP 房間 (區域網路自動發現)**
    *   預設連線方式。伺服器擷取連線者的公網 IP（對於 IPv4 私有網段如 `192.168.x.x` 或 `10.x.x.x`，伺服器會將其統一視為 `127.0.0.1`，歸入同一個 IP 房）。
    *   同一個 IP 房內的所有設備會自動互相顯示在首頁上，這模擬了 AirDrop 的區域網路發現功能。
2.  **配對房間 (Persistent Pairing — 跨網域/跨網路)**
    *   當兩台設備不在同一個區域網路內（例如一台使用家用 Wi-Fi，另一台使用手機 5G 訊號），可以使用**「配對功能」**。
    *   A 設備發起配對，伺服器會生成一個臨時的 6 位數配對碼/QR-Code，並關聯一個 256 位元的強隨機 `Room Secret`。
    *   B 設備輸入配對碼或掃描 QR-Code 後，兩台設備會同時加入該 `Room Secret` 的房間，完成握手。
    *   配對完成後，`Room Secret` 會永久儲存在雙方瀏覽器的 IndexedDB 中。每次開啟網頁時，兩台設備會自動向信令伺服器申報加入該私密房間。只要兩台設備同時在線，不論身處何地，都能直接連通。
3.  **臨時公共房間 (Temporary Public Rooms)**
    *   適合臨時分享。用戶可以主動創建一個由 5 個隨機字母組成的公共房間（例如：*abcde*）。
    *   其他用戶輸入該代碼後即可暫時進入此房間並與房內的所有人共享檔案。
    *   當關閉 PairDrop 網頁後，用戶會自動退出公共房間，安全性較高。

### 4.2 WebRTC 信令與 P2P 通道建立
當在主畫面上點擊某個 Peer 並選擇發送檔案時：
1.  發送端呼叫 `RTCPeerConnection`，並創建 `RTCDataChannel`。
2.  發送端生成一個 **Offer**，將其通過 WebSocket 發送給伺服器，伺服器依據 `to` 欄位轉發給接收端。
3.  接收端收到 Offer 後，呼叫 `setRemoteDescription`，生成 **Answer** 並回傳。
4.  雙方在握手過程中通過伺服器轉發交換 **ICE Candidates**（探測雙方的網路路徑，包括直接連線、經由 STUN 進行 NAT 穿透，或在極端對稱 NAT 網路下使用 TURN 轉發伺服器）。
5.  一旦 `RTCDataChannel` 的狀態轉為 `open`，WebSocket 連線即退居後台，檔案數據便開始以二進位分片形式直接在兩台設備間流動。

---

## 5. 與原版 Snapdrop 的改進與差異

PairDrop 作為 Snapdrop 的分支，解決了 Snapdrop 長久以來的許多痛點：

| 比較維度 | Snapdrop | PairDrop 的改進 |
| :--- | :--- | :--- |
| **跨網路傳輸** | 僅限於相同公網 IP (區域網路)。 | **支援配對（Pairing）與公共房間**，只要有網路，即使跨網域、跨國家、處於不同 NAT 後方也能互傳。 |
| **連線穩定性** | 在複雜的企業網、VPN、Apple Private Relay 環境下經常斷連或找不到設備。 | **內建自動配對、自動生成加密房間與健全的信令處理**，能更好地繞過 VPN 與 Private Relay 限制。 |
| **多檔案下載** | 傳送多個檔案時，接收端會彈出多次下載請求，非常混亂。 | **改進 UI 流程**。多檔案傳輸時會顯示全域進度條，傳輸完成後會自動使用 `zip.js` 將所有檔案**打包成一個 ZIP 壓縮檔**供用戶一鍵下載。 |
| **備用傳輸** | 如果 WebRTC 無法建立連線，傳輸直接宣告失敗。 | **支援 WebSocket 中轉 (WS Fallback)**。當 P2P 連線失敗時，可選擇透過伺服器 WebSocket 信道轉發數據，確保傳輸成功率。 |
| **行動端體驗** | 背景傳輸大檔案時，手機瀏覽器容易休眠導致傳輸中斷。 | 整合 `NoSleep.js`，在檔案傳輸期間**自動啟用 Wake Lock 防止螢幕與瀏覽器休眠**，傳輸完成後自動釋放。 |
| **系統整合** | 無法直接從系統選單分享。 | 支援 PWA 的 Web Share Target API，用戶可以直接在系統（Android/iOS）的**「分享」選單中選擇 PairDrop** 進行傳送。 |
| **部署便利性** | 後端需要 Node.js + Python (WS)。 | **純 Node.js 後端**，無 Python 依賴，體積更小，且支援 Docker 一鍵部署（內建與 Coturn TURN 伺服器的整合方案）。 |

---

## 6. 總結

PairDrop 是一個在架構設計上極致追求**輕量化**與**用戶隱私**的 Web 應用。
*   它利用 **WebRTC** 實現了高效且隱私的點對點檔案傳輸（檔案不留伺服器過路痕跡）。
*   它通過創新的 **LocalStorage 跨分頁管理** 與 **IndexedDB 設備持久配對** 機制，彌補了純 Web 應用在系統權限與持久連線上的短板。
*   代碼結構採用了良好的職責分離（SOC）原則，前端的延遲加載優化與事件驅動設計，也為現代 Web App 的性能與交互設計提供了極佳的參考範本。
