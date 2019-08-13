# Clos Network
Charles Clos曾經是貝爾實驗室的研究員。他在1953年發表了一篇名為“A Study of Non-blocking Switching Networks”的文章。文章裡介紹了一種用多級設備來實現無阻塞電話交換的方法，這是Clos的起源。

---
 
## Clos架構
簡單的Clos架構是一個三級互連架構，包含了輸入級，中間級，輸出級，如下圖所示：
![](https://i.imgur.com/QTYUbKh.jpg)
圖中的矩形是規模小得多的轉發單元，相應的成本小得多。Clos架構的核心思想是：**用多個小規模、低成本的單元構建複雜，大規模的架構**。上圖中，m是每個子模塊的輸入端口數，n是每個子模塊的輸出端口數，r是每一級的子模塊數，經過合理的重排，只要滿足```r2 ≥ max(m1,n3)```，那麽，對於任意的輸入到輸出，就能找到一條無阻塞的通路。

---

回到交換機架構，隨著網絡規模的發展，交換機的端口數量逐漸增多。crossbar模型的交換機的開關密度，隨著交換機端口數量N呈 $O(N^2)$ 增長。相應的功耗，尺寸，成本也急劇增長。在高密度端口的交換機上，繼續采用crossbar模型性價比越來越低。大約在1990年代，Clos架構被應用到Switch Fabric。應用Clos架構的交換機的開關密度，與交換機端口數量N的關系是 $O(N^{3/2})$ ，所以在N較大時，Clos模型能降低交換機內部的開關密度。這是Clos架構的第二次應用。

---

## Clos 網路架構
由於傳統三層網絡架構存在的問題，在2008年一篇文章A scalable, commodity data center network architecture，提出將Clos架構應用在網絡架構中。2014年，在Juniper的白皮書中，也提到了Clos架構。這一回，Clos架構應用到了數據中心網絡架構中來。這是Clos架構的第三次應用。

---

現在流行的Clos網絡架構是一個二層的spine/leaf架構，如下圖所示。spine交換機之間或者leaf交換機之間不需要鏈接同步數據（不像三層網絡架構中的匯聚層交換機之間需要同步數據）。每個leaf交換機的上行鏈路數等於spine交換機數量，同樣的每個spine交換機的下行鏈路數等於leaf交換機的數量。可以這樣說，spine交換機和leaf交換機之間是以full-mesh方式連接。

![](https://i.imgur.com/lvhbMqT.jpg)

---

可前面不是說Clos架構是三級設備架構嗎？為什麽這裏只有兩層網絡設備？這是因為前面討論Clos架構的時候，都是討論輸入到輸出的單向流量。網絡架構中的設備基本都是雙向流量，輸入設備同時也是輸出設備。因此三級Clos架構沿著中間層對折，就得到了二層spine/leaf網絡架構。由於這種網絡架構來源於交換機內部的Switch Fabric，因此這種網絡架構也被稱為Fabric網絡架構。

---

##### 在spine/leaf架構中，每一層的作用分別是：
* leaf switch：相當於傳統三層架構中的接入交換機，作為TOR（Top Of Rack）直接連接物理服務器。與接入交換機的區別在於，L2/L3網絡的分界點現在在leaf交換機上了。leaf交換機之上是三層網絡。
* spine switch：相當於核心交換機。spine和leaf交換機之間通過ECMP（Equal Cost Multi Path）動態選擇多條路徑。區別在於，spine交換機現在只是為leaf交換機提供一個彈性的L3路由網絡，數據中心的南北流量可以不用直接從spine交換機發出，一般來說，南北流量可以從與leaf交換機並行的交換機（edge switch）再接到WAN router出去。

---

##### 對比spine/leaf網絡架構和傳統三層網絡架構
![](https://i.imgur.com/cfpg5qr.jpg)

可以看出傳統的三層網絡架構是垂直的結構，而spine/leaf網絡架構是扁平的結構，從結構上看，spine/leaf架構更易於水平擴展。

### Spine-Leaf
![](https://i.imgur.com/d8jWRKB.png)

---

在以上兩級 Clos 架構中，每個低層級的交換機（leaf）都會連接到每個高層級的交換機 （spine），形成一個 full-mesh 拓撲。leaf 層由接入交換機組成，用於連接服務器等 設備。spine 層是網絡的骨幹（backbone），負責將所有的 leaf 連接起來。 fabric 中的每個 leaf 都會連接到每個 spine，如果一個 spine 掛了，數據中心的吞吐性 能只會有輕微的下降（slightly degrade）。

---

如果某個鏈路被打滿了，擴容過程也很直接：添加一個 spine 交換機就可以擴展每個 leaf 的上行鏈路，增大了 leaf 和 spine 之間的帶寬，緩解了鏈路被打爆的問題。如果接入層 的端口數量成為了瓶頸，那就直接添加一個新的 leaf，然後將其連接到每個 spine 並做相 應的配置即可。這種易於擴展（ease of expansion）的特性優化了 IT 部門擴展網絡的過 程。leaf 層的接入端口和上行鏈路都沒有瓶頸時，這個架構就實現了無阻塞（nonblocking）。

---

在 Spine-and-Leaf 架構中，任意一個服務器到另一個服務器的連接，都會經過相同數量 的設備（除非這兩個服務器在同一 leaf 下面），這保證了延遲是可預測的，因為一個包 只需要經過一個 spine 和另一個 leaf 就可以到達目的端。


資料參考:
https://zhuanlan.zhihu.com/p/29975418
https://arthurchiao.github.io/blog/spine-leaf-design-zh/
