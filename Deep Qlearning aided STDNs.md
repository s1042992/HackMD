# Deep Q-Learning Aided Networking,Caching, and Computing Resources Allocationin Software-Defined Satellite-Terrestrial Networks

## 題目動機

  使用軟體定義網路架構(SDN)下的衛星與地面網路(satellite-terrestrial networks (STNs))的趨勢已日益漸增，而實作STN要考慮的因素例如有：建網(Networking)，快取(caching)，運算(computing)，等等。本篇將這些聯合資源分配的問題視作聯合優化(joint optimization)的問題，但若是每次將這些因素進行運算的話，所耗費的空間與時間成本會太高，因此本篇透過DQN的方式來達到目的。

## 相關工作
* **衛星與地面之軟體定義網路(Satellite-Terrestrial Networks With Software-Defined Networking (SDSTNs))：**
  作者認為若使用一般常見的TCP/IP 無法滿足衛星地面網路之需求，因此想要使用SDN的架構。
  
* **衛星與地面網路之快取(Satellite-Terrestrial Networks With Caching)：**
  因為傳輸延遲高，資料高錯誤，斷斷續續的連接以及高移動性，作者認為要解決這些問題必須使用網路內快取(in-network cashing)。

* **衛星與地面網路之邊際運算(Satellite-Terrestrial Networks With Edge Computing)：**
  因為衛星的運算能力有限，若採取雲端運算也不是好選擇，因為要利用衛星傳送太多資料。因此作者在本篇採取邊際運算。
  
## 系統架構與模型

  SDSTN分三層，第一層為資料層，共有三種基本設備：網路設備(LEOs), 快取設備(Content caches), 運算設備(MEC servers)。第二層為控制層，由MEO、GEO與基地控制台共同組成一個控制系統。而控制層也是本架構最主要的核心，他的責任是控制與管理多個資料層的資源。根據使用者的需求以及考量目前資源池情況下，動態的分配資源給使用者。最後的第三層為應用層，顧名思義跟衛星有關的應用都是在這一層，例如監視，感測，通訊，等等。
  
  ![](https://i.imgur.com/6cgODhE.jpg)

### 工作流(workflow):

  假設一用戶需要一個衛星影像，會先將需求發給範圍有覆蓋到用戶的LEO，LEO會去確認用戶所要的資料是否已在LEO上的cache中，若是已經在cache中，則再去確認版本是否與用戶裝置一致，若一致，則回傳內容給用戶，若版本不同則會建立一個轉碼(transcoding)任務給MEC，待轉碼完畢後再將內容傳回給用戶。若一開始內容就不在所屬的cache裡的話，會去確認所有content caches是否有所需內容，若有則執行剛上述的檢查動作，若所有caches都沒有此內容，LEO會請所屬衛星進行感測或拍攝影像，傳回給用戶並判斷此內容是否需要留在caches中。
  ![](https://i.imgur.com/GbRokdo.jpg)
  
  
### 覆蓋時間模型(Coverage Time Model)：
  先計算出用戶與衛星可傳輸狀態下之最大角度
  ![](https://i.imgur.com/isDhsv2.jpg)
  目的要推測在此角度A時，下一個時間點在角度B的機率，並可以計算出LEO覆蓋用戶的可能時間。
主要透過馬爾可夫模型得到馬爾可夫鏈進而得到此機率矩陣:
![](https://i.imgur.com/V0ET9Vk.jpg)
主要就是記錄事件在這個狀態A下發生狀態B的機率為何。

### 通訊模型(Communication Model):
  與上面概念相似，也是根據目前SNR的狀態來推測下一個SNR的狀態，
![](https://i.imgur.com/umGOP8u.png)
並且可以進一步計算出通訊率(communication rate):
![](https://i.imgur.com/eMBUyv7.png)
$v_u^l(t)$代表用戶在時間t時的頻譜效率。
$a_u^l(t)$代表用戶是否使用LEO的網路資源 1代表有, 0代表沒有。

### 快取模型：
  將資料根據使用頻率進行排序，根據齊夫定律(Zipf slope)，可得知第二名的使用頻率是第一名的1/2，第三名的使用頻率是第一名的1/3…..
因此每個資料的取用機率可以表示為：

$λ_i(t)=\frac{β}{π^\alpha}$

β為松柏過程(Poisson process)的參數。
那預測的狀態機率矩陣也與剛剛相同的概念：

![](https://i.imgur.com/nlPymsk.png)

### 計算模型:
  計算完成每個任務所需花的CPU cycles數，與之前的模型概念相同，也就是推算下一個任務所需要花的cycles數之機率。
  
![](https://i.imgur.com/Vy3cQqu.png)

並且可以進一步得知計算率(computation rate):

![](https://i.imgur.com/6chbjno.png)

最後整合，得到龐大的狀態空間：

![](https://i.imgur.com/KSx7TOq.png)

### 動作空間:
  ![](https://i.imgur.com/aSr2hBN.png)
  
### 獎勵函數：
  ![](https://i.imgur.com/gRpyjve.png)
  
狀態空間太大，計算複雜，耗時長，這就是作者要使用DQN來取代QN的原因。

使用DQN方法後，關鍵處變成:

![](https://i.imgur.com/MtA2D14.png)

簡化很多，並且不用一直更新Q-table，只要去更新權重就好。

最後作者實驗結果認為DQN可以有效得解決動態分配多資源的問題。

### 參考文獻
1.	A.Vanelli-Coralli,G.E.Corazza,M.Luglio,andS.Cioni,“TheISICOM architecture,” in Proc. Conf. Satell. Space Commun., pp. 104–108(2009).
2.	A. Armon and H. Levy, “Cache satellite distribution systems: Modeling, analysis, and efﬁcient operation,” IEEE J. Sel. Areas Commun., vol. 22, no. 2, pp. 218–228, Feb. 2004.
3.	S.G.Chang,“Caching strategy and service policy optimization in a cache satellite distributio nservice,”Telecommun.Syst.,vol.21,no.2–4,pp.261– 281, 2002.
4.	L. Galluccio, G. Morabito, and S. Palazzo, “Caching in informationcentric satellite networks,” in Proc. IEEE Int. Conf. Commun., 2012, pp. 3306–3310.











