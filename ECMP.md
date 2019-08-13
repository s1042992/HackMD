# 等價多路徑路由 ECMP
等價多路徑路由，即存在多條到達同一個目的地址的相等開銷的路徑。當設備支持等價路由時，發往該目的IP 或者目的網段的三層轉發流量就可以通過不同的路徑分擔，實現網絡鏈路的負載均衡，並在鏈路出現故障時，實現快速切換。

----

![](https://i.imgur.com/yQVhtXF.jpg)
ECMP流程圖

----

目前資料中心網路(Data Center)廣泛應用的Fabric架構中會應用大量的ECMP，其優點主要體現在可以提高網路冗餘性和可靠性，同時也提高了網路資源利用率;大量的ECMP鏈路在特定場景下執行過程中會引發其他問題。例如，當某條ECMP鏈路斷開後，ECMP組內所有鏈路流量都會被重新HASH，在有狀態的伺服器區域(如LVS)中將導致雪崩現象，又或者會出現多級ECMP的HASH極化導致鏈路擁塞等。

---

### 步驟一：HASH因子的選擇
首先數據報文轉發查詢路由表，確認存在多個等價路由，**再根據當前用戶配置的流量均衡算法**，提取參與 HASH 計算的關鍵字段，即HASH因子。ECMP 流量均衡可選擇的 HASH 因子如下表：
![](https://i.imgur.com/9JhIdH0.jpg)

----

### 步驟二：HASH計算
基於步驟一提取的 HASH 因子，根據 HASH 演算法進行計算，得出相應的 HASH lb-key(load-balance key)。 ECMP 流量均衡支援的 HASH 演算法包括異或(XOR)、CRC、 CRC+擾碼等。

----

HASH演算法有很多種，我們以XOR演算法為例做出說明。
XOR運演算法則為兩個輸入位元位相同時為0，不同則為1。
HASH因子不同，運算結果也不盡相同。

----

1. HASH因子为IP address source(SIP)：
```
    a. SIP XOR 0 ，得出一个32bit的數值a
    b. 將數值a再進行高16bit和低16bit做XOR計算得出16bit數值b
    c. 數值b的15~12bit與11~8bit再做XOR計算，得出4bit數值c
    d. 數值c替換數值b的11~8bit，得出數值d
    e. 數值d擷取低位10bit即為lb key
``` 

----

2. HASH因子为SIP+DIP/DIP：
```
    a. DIP XOR SIP ,得出一個32bit的數值a
    b. 剩餘運算步驟與SIP運算一致
```

----

3. HASH因子为SIP+DIP+SP+DP：
```
    a. SIP XOR DIP得到32bit的数值a
    b. 數值a的低16bit XOR SP 得到32bit的數值b
    c. 數值b的低 16bit XOR DP 得到 32bit 的數值c
    d. 數值c的高16bit XOR 低16bit得到16bit的數值d
    e. 數值d的15～12bit XOR 11～8bit，得到4bit的數值e
    f. 數值e替換數值d的11～8bit，得出數值f
    g. 數值f擷取低10bit，即為lb-key
```

---

### 步驟三：確認轉發下一跳(next hop)
資料報文經過路由查表後找到對應ECMP 基值(base-ptr),根據 HASH 因子通過 HASH 演算法計算獲得 HASH lb-key 後，進行 ECMP 下一跳鏈路數(Member-count)求餘計算，再與ECMP基值進行加法運算得出轉發下一跳index，即確定了下一跳轉發路由。

```
計算公式：Next-hop =(lb-key % Member-count)+ base-ptr
```

---

## ECMP 問題
上述流程為ECMP常規轉發流程，但在特定網路環境下執行過程中就會出現問題，接下來繼續分析資料中心網路中ECMP遇到的2個常見問題。

----

### 單鏈路故障導致ECMP組所有資料流被重新HASH計算
當Leaf交換機發送6條資料流到LVS伺服器(Linux Virtual Server)，Leaf先進行HASH運算負載均衡到每一臺LVS伺服器上，正常流量轉發如圖所示：

![](https://i.imgur.com/T4dYxJk.jpg)

----

當某臺LVS伺服器網絡卡出現故障或者鏈路出現故障，Leaf交換機會將ECMP組內資料流將重新HASH計算，再進行負載均衡到剩餘有效鏈路上，進而導致TCP會話斷開，發生雪崩現象，例如一些支付類業務，同一個使用者的一次支付過程會呼叫多個業務服務，業務側要求一次支付的過程都落在同一個處理伺服器上，當出現單條鏈路故障後不僅影響該鏈路所在LVS承載的使用者，同時還影響該ECMP組下其他LVS承載的使用者，如圖例所示：

----

![](https://i.imgur.com/ynjI7Z2.jpg)

---

### HASH極化問題
在Leaf裝置和Spine裝置均採用上聯鏈路數為偶數且**ECMP演算法**及**HASH因子一致**的情況下，資料流在Leaf裝置上經過一次HASH計算，將資料流負載分擔到兩臺Spine上，均衡後效果為資料流1、2、3轉發至Spine-1，資料流4、5、6轉發至Spine-2，Spine再進行HASH計算負載分擔到兩臺DCI核心上，因在Spine層採用的HASH演算法與Leaf的HASH演算法一致，最終Spine-1的資料流1、2、3均轉發至DCI-1上，未負載分擔到DCI-2上任何資料流，而Spine-2的資料流4、5、6均轉發至DCI-2上，未負載分擔到DCI-1上任何資料流，同理Leaf-2傳送的資料流也是如此，進而產生HASH極化問題，導致SPINE和DCI之間鏈路有一條空閒，極大的浪費了網路資源，甚至會導致流量擁塞。

----

![](https://i.imgur.com/mon9RRG.jpg)

----

#### HASH極化-優化方案
* 同廠商Leaf裝置和Spine裝置均採用相同上聯鏈路數場景下，應避免在相鄰的兩臺裝置上使用相同的負載均衡演算法
* 裝置在執行HASH計算時，除傳統的五元組外，可以增添擾動因子，避免HASH計算結果相同。
* HASH擾動的計算過程中HASH因子仍然正常提取，再增加使用者自定義隨機擾動因子，經過HASH演算法運算時，不同交換機HASH計算結果就將不一致，以達到避免HASH極化現象的出現。

---

## 動態負載均衡技術實現
在資料中心網路中，突發流量多，並且存在大象流和老鼠流並存現象，本文所描述的基於資料流五元組的HASH演算法，並結合HASH擾動因子技術實現流量負載均衡，但無法實現大象流和老鼠流並存的網路中多鏈路之間的流量負載均衡。










