# 2020/03/13 P4 RLNC_redundancy
###### tags: `P4` `implementation`
## Introduction
此次實驗承接上周的RLNC實驗，這次主要解決上次實驗產生的問題，所以會修改NC的機制與編解碼範圍。
## Implementation
這次實驗需要實現幾個目標：
1. bmv2中編碼與解碼的register存取機制需個別獨立，解碼時的register存取需使用header中的label來進行批次處理，防止順序亂掉時發生錯誤，只是此作法需建立新的register來儲存每個批次的處理進度。
2. NC的方式改成跨session的編碼，編碼後的封包(包括redundancy)就不須考慮ip header的內容，不會有forwarding的問題。
3. 將編碼範圍增大至包含整個tcp header的內容，編碼後的封包可以完全替代彼此，只要收到**任何**足夠數量的編碼封包都能還原封包，這時候解碼就不須辨識封包是什麼session，可以將NC header中的flowID移除。
### NC header
修改後的header如下
![](https://i.imgur.com/56danO3.png =350x)
### NC packet
目前NC packet如下
![](https://i.imgur.com/yJOol6l.png)

### flowchart
實現後bmv2的處理流程如下：
![](https://i.imgur.com/sfWJpta.jpg)

## Experiment design
### topology
設計拓樸如下，拓樸為兩個session的TCP傳輸，透過`h1`→`h2`發送`P1`、`P2`封包，在經過S1時除了需進行原本數量的線性組合編碼外，還需多產生redundancy，在經過S2時需進行解碼並轉發回`h2`，當S2收到多餘的redundancy必須將其丟棄。
![](https://i.imgur.com/te17ANz.png)

### result
* 不管鏈路是否完整，只要收到足夠數量封包就能還原。
![](https://i.imgur.com/lKfOsIS.png)
* 觀察s3-s2之間的鏈路，紅色框＝NC header，黑色框＝tcp header+payload，可以發現tcp內容全部經過編碼無法辨識。
![](https://i.imgur.com/QVrm1pb.png)
* 觀察s4-s2之間的鏈路，紅色框＝NC header，黑色框＝tcp header+payload，可以發現tcp內容全部經過編碼無法辨識。
![](https://i.imgur.com/qAQWxni.png)

* 觀察s5-s2之間的鏈路，紅色框＝NC header，黑色框＝tcp header+payload，可以發現tcp內容全部經過編碼無法辨識。
![](https://i.imgur.com/sTNQCUM.png)

## Problem
編碼範圍增大導致程式碼偏大，因為每次編解碼只能1個byte，在P4無迴圈的環境中，當封包過大時，會產生這個問題。如果使用resubmit製造循環，雖然可以減少程式碼數量，但會導致運算速度過慢(Multi-thread只能運作不同的switch模組，某個pipeline thread的開始必須等上個thread的結束)，且會影響原本編解碼流程。