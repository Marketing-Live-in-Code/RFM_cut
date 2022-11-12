# RFM顧客分類模型實戰教學【附Python程式碼】

> 便利商店、超市、量販店畫出來的RFM一樣嗎？

自從寫完[「原來用Python實作行銷RFM model，可以那麼簡單！-【附Python程式碼】」]()這篇文章後，有不少朋友問我：「所以RFM切割的依據是什麼？」，以前我總是回答「要看產業」、「你應該比我了解」、「大概抓一個區間」。後來我想想，以資料科學的角度，決策必須有憑有據，不能全靠直覺判斷，因此本篇文章，將以科學的角度，告訴您「怎麼切割RFM比較好」。

切割RFM資料也是一門藝術，絕對不可能是平均的切割，因為每個不同的產業，所對應的顧客購買間隔是不同的，甚至同一群顧客。由於每個顧客有自己的購買時間軸，如何精準的找到每一類顧客，購買時間的分水嶺，是我們今天要解惑的議題。

## 您可能要先知道RFM基本實做：
> Python：
> [「原來用Python實作行銷RFM model，可以那麼簡單！-【附Python程式碼】」]()

> R語言：
> [「常貴客？新客？ 讓RFM模型簡簡單單解釋一切！（附實現程式碼）」]()

![三種零售業差距](https://i.imgur.com/W326NcS.png)
在進行切割前，必須先了解，您的商店有什麼特性，就算是零售業也大不同。以下圖一為例，請問便利商店、超市、量販店有什麼不同？用賣的商品來區分，肯定很難區分。而您也很常看到某間量販店的旁邊，還有兩三間便利超商（在台北更常見），如果他們沒有不同，早就互相競爭而倒閉了。

> 要從「解決消費者需求」的角度出發：
> * 便利商店：解決即時所需。
> * 超 市：解決一日所需。
> * 量販店：解決一週所需。

由此可知，本質上三種零售商店的顧客購買頻率就有很大的差別，因此若都以固定的切割模式，或時間軸來繪製RFM，是非常不合理的。就從統計學的角度來決定切割吧！

## 切割原理
切割的原理，其實是利用簡單的統計參數，作為切割的依據。我們會計算出每個顧客每次購買時的間隔，把這些間隔的時間做統計分析，將算出的百分位距來作為切割的依據。
![盒狀圖](https://i.imgur.com/n8P5CRg.png)
如上圖所示，假設要將RFM資料切分成4份，那就依照消費者的購買間隔天數，找到25百分位、50百分位、75百分位，作為切割的標準。為何要以此來區分呢？因為這符合RFM的設計原理，依照消費者的購買頻率來觀測收入，又能體現RFM無法觀測到的：流失客問題，可能該顧客很久以前是常貴客，但它其實它現在已經換了別間店消費，在分析上該顧客會變成常貴客。
![經由百分位切割的結果](https://i.imgur.com/Y43tefO.png)

## Python實做
![完整程式碼畫面](https://i.imgur.com/YXSCmKj.png)
首先利用pivot_table方法，計算出每個消費者，每次消費時，分別購買幾樣商品。另一方面，也可以順便統整該名消費者，到底來消費幾次。
```python
orders['values'] = 1
purchase_list = orders.pivot_table(
        index=['clientId','gender','orderdate'], #分類條件
        columns='product', # 目的欄位
        aggfunc=sum, # 計算方式，max, min, mean, sum, len
        values='values' #根據欄位
        ).fillna(0).reset_index()
```
![執行完成pivot_table後結果](https://i.imgur.com/fWTM0qK.png)
因為資料的時間是2017年，因此我們以資料最近的那天（2017/4/17）作為分析的基準日期，以就是以那天來進行相減，算出RFM中所需要的recency。
```python
#設定今天的日期為最近一位顧客購買的日期，從那天來看過往的銷售狀況
theToday = datetime.datetime.strptime(orders['orderdate'].max(), “%Y-%m-%d”)# 將購買清單資料中'orderdate'的欄位，全部轉換成datetime格式
purchase_list['orderdate'] = pd.to_datetime(purchase_list['orderdate'])# 計算消費者至今再次購買與上次購買產品的時間差'
purchase_list['recency'] =( theToday — purchase_list['orderdate'] ).astype(str)# 將'recency'欄位中的days去除
purchase_list['recency'] = purchase_list['recency'].str.replace('days.*', #想取代的東西
                                  '', #取代成的東西
                                  regex = True)
將'recency'欄位全部轉換成int
purchase_list['recency'] = purchase_list['recency'].astype(int)
```
最後用pandas套件方法groupby，其中的diff()方法，它能幫我們計算每筆資料的數值間隔。
```python
purchase_list['interval'] = purchase_list.groupby(
                               “clientId”, #分類條件
                               as_index = True # 分類條件是否要取代Index
                               )['orderdate'].diff()purchase_list.dropna(inplace = True)#刪除第一次來本店的資料
purchase_list['interval'] = purchase_list['interval'].astype(str) # 將時間資料轉成字串
purchase_list['interval'] = purchase_list['interval'].str.replace('days.*', '').astype(int) #將欄位中的days去除
```
![間隔計算完成](https://i.imgur.com/bTUEY1s.png)
到這裡所需的資料都完成了，接下來就是利用統計分析，看看消費者間隔天數（interval）的資料差異如何。可以利用pandas套件的describe()方法，列出interval欄位的總數（count）、平均（mean）、標準差（std）、最小值（min）、25百分位（25%）、50百分位（50%）、75百分位（75%）、最大值（max）。
![describe方法執行結果](https://i.imgur.com/ro3xkLx.png)
結果可以發現，間隔平均只有19.45天，但間隔最久的顧客甚至到達89天，由此資料繪製程圖7的圖形，可以發現整個資料大的非常極端，但其實這些資料在RFM中應該可以全部匯集成一個區塊消費者，因為這些消費者數量不多。
![將interval欄位畫成盒狀圖](https://i.imgur.com/NtV9RMv.png)
獲取百分位數必須使用quantile方法，在參數中以陣列輸入想要的百分位即可。例如輸入[0.25, 0.5, 0.75]，就會得到結果6、14、27。因此若要將RFM切割成四等分，就能以以上三個數字做切割。
![quantile執行範例](https://i.imgur.com/2Px564k.png)
```python
purchase_list['interval'].quantile([0.25, 0.5, 0.75])
```
我們以往的RFM切割，都是切成6X6，因此這裡將其區分成6份，因為切分成6份會無法整除，因此大概以16百分位為級距。最後得結果4、9、14、21、35。
```python
purchase_list['interval'].quantile([0.16, 0.32, 0.5, 0.66, 0.82])
```
![切割6X6依據天數](https://i.imgur.com/ezhDQdF.png)

## 管理意含
> 數據可載企業，亦能覆企業

有依據的切割後結果，與前者做比較。可以發現，最明顯的差距是量販客的部份。因為量販客是最難區隔出來的客群，因為要同時符合高frequency與recency的條件，若有一邊的條件設定不佳，都會導致量販客會有空洞（如下圖左半部）。

如果您是老闆或高階主管，在屬下向您報告RFM分析結果時，必須問他切割的依據。經由這項調整後，會發現這間門市的量販客其實也不少；因此整個切割的小變化，會引導整個決策的風向。
![調整結果前後比較](https://i.imgur.com/TKZmvQv.png)

