# Risk Box：金融通訊協議層聯防架構白皮書

![Version](https://img.shields.io/badge/Version-v2.0.0--Enterprise-blue)
![Status](https://img.shields.io/badge/Status-Stable-green)
![Layer](https://img.shields.io/badge/Layer-L7--App--Parsing-orange)

Risk Box 是一款部署於 **OSI L7 (應用層)** 的金融風險監控與即時阻斷節點。透過高性能協議解析與跨行聯防機制，在不侵入銀行核心系統的前提下，提供毫秒級的詐騙偵測與交易干預。

---

## 1. 系統定位 (System Positioning)

* **非侵入架構**：採用 eBPF 內核追蹤或 TAP 旁路監聽，對 Core Banking 零干擾。
* **深度協議解析 (DPI)**：原生支援 ISO-8583 (Card/ATM) 與 ISO-20022 (Swift/Real-time Payment)。
* **主動防禦機制**：支援 L4 精準 Session Reset 與 L7 應用層拒絕響應。
* **隱身部署**：支援無 IP 位址監控模式 (NO-IP)，避免掃描攻擊。

---

## 2. 協議解析引擎 (Protocol Engine)

系統將二進制流量即時還原為金融特徵向量，以下為關鍵維度映射表：

| 監控維度 | ISO-8583 (ATM/POS) | ISO-20022 (匯款/清算) |
| :--- | :--- | :--- |
| **交易金額** | DE-4 (Amount) | `IntrBkSttlmAmt` |
| **傳輸時間** | DE-7 (Transmission D/T) | `CreDtTm` |
| **商戶類別** | **DE-18 (Merchant Type)** | `SplmtryData/.../MerchantCategoryCode` |
| **收單機構** | DE-32 (Acquiring Inst ID) | `InstgAgt/FinInstnId/BICFI` |
| **受款帳戶** | DE-102 (Account ID To) | `CdtrAcct/Id/Othr/Id` |
| **付款帳戶** | DE-103 (Account ID From) | `DbtrAcct/Id/Othr/Id` |
| **終端識別** | **DE-41 (Terminal ID)** | `Envlp/TermnlId` |

---

## 3. 風險演算模型 (Risk Scoring Model)

### 3.1 核心公式 (R_score)
為確保模型穩定性並防止權重異常溢出，採用 **Sigmoid 函數** 進行歸一化處理：

$$R_{score} = \frac{1}{1 + e^{-(\sum (W_i(t) \cdot \delta_i) + \Gamma - \theta)}}$$

* $W_i(t)$：動態時間權重係數。
* $\delta_i$：特徵偏離值（Deviation）。
* $\Gamma$：聯防共振因子（由跨節點共識觸發）。
* $\theta$：系統靈敏度閾值偏置。

### 3.2 信心度修正 (C)
信心度隨回報節點數量 $N$ 非線性遞增，確保集體決策的可靠性：

$$C = \frac{\sum_{j=1}^{N} (T_j \cdot S_j)}{\ln(e + N_{rep})} \cdot \alpha$$

---

## 4. 風險評分維度矩陣

### 4.1 個體風險特徵 (Behavioral DNA)
| 編號 | 維度 | 說明 | 權重 |
| :--- | :--- | :--- | :--- |
| **R1** | 高齡異常交易 | >70歲帳戶 + 偏離歷史頻率 | +40 |
| **R4** | 異常地理跳躍 | 同帳戶短時間跨區域存取 | +30 |
| **R6** | 快速金流外移 | 資金轉入後 5 分鐘內即轉出 | +45 |

### 4.2 群集拓撲風險 (Graph Topology)
| 編號 | 維度 | 說明 | 權重 |
| :--- | :--- | :--- | :--- |
| **G1** | 同裝置多帳戶 | 相同 DE-41 終端 ID 關聯多組帳戶 | +50 |
| **G5** | 分層清洗行為 | 金流拓撲呈現 3 層以上鏈式轉帳 | +55 |

---

## 5. 四級風險警示與干預機制

| 等級 | R_score 區間 | 處置行為 | 主動干預手段 (Active Mode) |
| :--- | :--- | :--- | :--- |
| **Green** | $< 0.60$ | 正常交易 | 無 (放行並記錄) |
| **Yellow** | $0.60–0.80$ | 風險標記 | 增強型日誌採樣 |
| **Orange** | $0.80–0.95$ | 二次確認 | 注入毫秒級延遲 (Latency Injection) |
| **Red** | $> 0.95$ | **即時阻斷** | **L4 精準 TCP RST / L7 拒絕碼注入** |

---

## 6. 聯防架構 (P2P Gossip Network)

* **隱私保護**：採用 **Salted-HMAC-SHA256** 對帳號進行脫敏，僅交換行為特徵雜湊。
* **共識機制**：基於 Raft 狀態機原理，達成全網黑名單秒級同步。
* **共振觸發**：當 3 個以上信任節點回報同一特徵為 Red，$\Gamma$ 自動激發，強制拉升該交易風險等級。

---

## 7. 效能與部署規格

### 7.1 工程規格
* **處置延遲**：解析 + 推論全流程 $< 10ms$。
* **吞吐量**：單節點 $\ge 3,000$ TPS。
* **高可用性**：支援 Hardware Bypass Switch，物理層斷電自動導通。

### 7.2 部署模式
1.  **Observer**：透過 TAP 接入，零延遲、零風險。
2.  **Marked**：Inline 透明橋接，進行實時標記與風險預警。
3.  **Active Defense**：正式開啟協議層阻斷能力，形成即時防線。

---

## 8. 法律與合規性

* 符合 **GDPR / 金融個資法**：不儲存報文全文，僅保留非去識別化之特徵向量。
* **審計追蹤**：所有干預動作均附加數位簽章，滿足監管審計需求。

---

## 結語

Risk Box 透過修正後的 L7 協議映射與穩定的數學模型，為現代金融體系提供了一套高可靠、低延遲的即時風控方案。
