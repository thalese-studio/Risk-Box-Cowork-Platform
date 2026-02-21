# Risk Box 技術架構規範 (Technical Architecture)

> **版本：** v1.0.0-alpha  
> **狀態：** 研究發表 (Research Publication)  
> **核心目標：** 透過物理層 (L2) 偵測與 ISO-8583 演算，建立即時斷流防線。

---

## 1. 系統定位：物理層防禦節點 (L2 Defense Node)

不同於傳統運行在應用層 (L7) 的防火牆，**Risk Box** 運作於數據鏈路層 (Data Link Layer)。

### 1.1 透明模式與隱身特性
* **無 IP 部署**：設備運作時不分配 IP 位址，對外部網路掃描完全隱身。
* **物理 Bypass (Fail-to-Wire)**：採用硬體級電路，確保設備斷電或異常時，金融電文流動不中斷，消除銀行部署疑慮。

---

## 2. ISO-8583 實時解析引擎

Risk Box 直接針對金融交易標準 **ISO-8583** 進行封包深度檢查 (DPI) 與欄位拆解。

### 2.1 關鍵監控數據元 (Data Elements)
我們針對以下欄位進行行為權值計算：

* **DE-4 (Amount)**：偵測異常金額波動。
* **DE-7 (Time)**：計算交易頻次與速率。
* **DE-18 (Merchant Type)**：識別高風險商戶類別。
* **DE-43 (Location)**：地理空間位移邏輯衝突檢查。
* **DE-102 (Account ID)**：行為 DNA 雜湊生成之唯一識別碼。

---

## 3. 風險演算模型 (Calculation Model)

### 3.1 風險權值得分 ($R_{score}$)

系統對每一筆電文進行多維度矩陣運算。公式定義如下：

$$R_{score} = \sum_{i \in DE} (W_i(t) \cdot \delta_i) + \Gamma$$

* **$W_i(t)$**：動態時間權重（基於近期詐騙模式）。
* **$\delta_i$**：行為偏離值（相對於該帳戶之歷史基準線）。
* **$\Gamma$**：聯防共振因子（跨節點同步之風險加成）。

### 3.2 信心度指數 ($C$)

為防止惡意節點進行數據投毒，所有協作回報需經過信心度過濾：

$$C = \frac{\sum (T_j \cdot S_j)}{\sqrt{N_{rep} + \epsilon}}$$

* **$T_j$**：節點信任等級。
* **$S_j$**：特徵空間分布相似度。
* **只有當 $C > \text{Threshold}$ 時，該特徵才會正式寫入聯防黑名單。**

---

## 4. 斷流機制 (The Kill Switch)

當 $R_{score}$ 突破閾值（預設為 80），Risk Box 將執行：

1. **即時丟棄 (Packet Dropping)**：在電文抵達核心系統前阻斷。
2. **特徵廣播 (P2P Gossip)**：將風險雜湊同步至聯防網路。
3. **證據封封存**：將觸發數據存入不可竄改日誌。

---

## 5. 隱私防護與合規 (Privacy & Compliance)

* **Non-PII 原則**：不儲存明文卡號或姓名，所有帳戶識別皆轉換為單向雜湊。
* **同態運算概念**：在不觸及隱私敏感數據的情況下，進行行為模式對比。

---

## 6. 部署拓撲 (Deployment Topology)

```mermaid
graph TD
    A[ATM/Digital Channel] --> B{Router/Switch}
    B --> C[Risk Box Node - L2 Mode]
    C --> D[Bank Core System]
    C -.-> E[Global Collaboration Network]
    
