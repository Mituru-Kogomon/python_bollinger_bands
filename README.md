
# 株価は95.4%の確立でボリンジャーバンド±2σの範囲内に収まる

For my python study. Stock trading with Bollinger Bands.

**仮説** 4.6%でしか負けないならボリンジャーバンドで売買すれば絶対に勝てる

**条件** 初期資本100万円、1ポジション100株、手数料0、副ポジション無し、25日移動平均線を基準

## 使用ライブラリ
- yfinance 株価取得
- Pandas データフレーム
- matplotlib.pyplot グラフ
- tqdm プログレスバー
- datetime Timestampオブジェクト
- os csv保存

## 処理順序
1. 株価取得
2. 移動平均線、 標準偏差、ボリンジャーバンド、乖離率の算出
3. 売買ルール制定、バックテスト
4. リターンの算出
5. グラフ化

## 必要なライブラリのインストール
```python perl:install
    pip install yfinance
    pip install pandas
    pip install matplotlib
    pip install tqdm
```
## 株価取得
(株)みずほフィナンシャルグループ【8411】を取得する
```perl:取得する銘柄コードを格納
    Scraping_Stocks = [
        "8411",
    ]
```
```perl:data_listに株価取得
    import yfinance
    import pandas as pd
    from tqdm import tqdm

    data_list = []
    for code in tqdm(Scraping_Stocks):
        tmp = yfinance.download(f"{str(code)}.T", progress=False)
        tmp["code"] = code
        data_list.append(tmp)
```
データフレームを結合し、中身を確認する
```
    df = pd.concat(data_list) #データフレームの結合
    df #データの確認
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/6a861925-34ae-322b-89e1-08cf0a911cd0.png)  
20年分の株価データを取得することが出来た  
調整後終値をプロットする  
```perl:"Adj Close" = 調整後終値
    import matplotlib.pyplot as plt
    df["Adj Close"].plot()
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/18158af2-5539-1738-2c18-66755458bc2c.png)  
2007年頃をピーク、2012年がボトムで、直近上昇傾向  

:::note info  
調整後終値 = 株式分割前の終値を株式分割後の終値に調整したもの  
:::  

## 移動平均線、 標準偏差、ボリンジャーバンド、乖離率の算出
```perl:25日移動平均線を基準とする
    window = 25  # 移動平均のウィンドウサイズ
    df['MA'] = df['Adj Close'].rolling(window=window).mean() #移動平均線
    df['StdDev'] = df['Adj Close'].rolling(window=25).std() #標準偏差
    df['Deviation'] = (df['Adj Close'] - df['MA']) / df['StdDev'] #移動平均線からの乖離率
    df['Bollinger bands+2'] = df['MA'] + (2*df['StdDev']) #ボリンジャーバンド+2σ
    df['Bollinger bands-2'] = df['MA'] - (2*df['StdDev']) #ボリンジャーバンド-2σ
```
データを視覚的に確認する
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/ab4a7de4-1147-f776-5490-950f634bc903.png)  
移動平均線が算出されるまでは NaN値 となっている  

20年分だとグラフが潰れて確認できない。  
ロシアのウクライナ進行付近、2022年1月から2023年6月までを範囲指定しグラフ化する  
```perl:プロットの範囲指定
    start_date = '2022-01-01'
    end_date = '2023-06-06'

    # 範囲指定のためのスライシング
    filtered_data = df.loc[start_date:end_date]
```
**標準偏差**
```perl:データのプロット
    filtered_data['StdDev'].plot()
    plt.show() # プロットの表示
```
戦争よりも金融危機のほうが反応している、直近は40円程度の利確は積極的に狙えそう  
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/010d7476-05f3-35aa-6501-dad2336605c8.png)  
**調整後終値、移動平均線、ボリンジャーバンド**
```perl:データのプロット
    filtered_data[['Bollinger bands+2','MA','Bollinger bands-2','Adj Close']].plot()
    plt.show() # プロットの表示
```
ほぼボリンジャーバンドの内側に収まっているように見える
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/43228ab3-e07f-15b6-cd6c-7e88d422ff52.png)  
**移動平均線からの乖離率**
```perl:データのプロット
    filtered_data[['Deviation']].plot()
    plt.show() # プロットの表示
```
±2σの範囲外で取引が終わる日数 522日間 × 4.6% = 24日間  
24日間よりも多く見える...と思う  
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/1d644a41-00c3-de48-d01a-fbee3117ccaa.png)  

## 売買ルール制定、バックテスト
**売買ルール**
* cash100万円
* 1lot100株
* 手数料0
* ポジションを有していたら反対売買のみ行う
```perl:今日をTimestampオブジェクトに変換
    import datetime
    today = datetime.date.today()
    today = pd.Timestamp(today)  # 日付をTimestampオブジェクトに変換
```
```perl:売買シグナル、ポジション変数を初期化
    df['Signal'] = 0
    df['Position'] = 0
```
```perl:バックテストの実行
for i in range(25, len(df)): #24行目以前は 移動平均線等が NaN のため25行目から最終行まで
    if df.index[i] == pd.Timestamp(today.date()):
        break

    df.loc[df.index[i], 'Position'] = df.loc[df.index[i-1], 'Position']
    if df['Adj Close'].iloc[i] <= df['Bollinger bands-2'].iloc[i]:
        if df['Position'].iloc[i] == 0:  # ポジションがない場合にのみ買いシグナルを発生
            df.loc[df.index[i], 'Position'] = 1
            df.loc[df.index[i], 'Signal'] = 1
        elif df['Position'].iloc[i] == -1:  # 売りポジションがある場合には売りポジションをクローズ
            df.loc[df.index[i], 'Position'] = 0
            df.loc[df.index[i], 'Signal'] = 1
    elif df['Adj Close'].iloc[i] >= df['Bollinger bands+2'].iloc[i]:
        if df['Position'].iloc[i] == 0:  # ポジションがある場合にのみ売りシグナルを発生
            df.loc[df.index[i], 'Position'] = -1  # ポジションを売りに切り替え
            df.loc[df.index[i], 'Signal'] = -1
        elif df['Position'].iloc[i] == 1:  # 買いポジションがある場合には買いポジションをクローズ
            df.loc[df.index[i], 'Position'] = 0
            df.loc[df.index[i], 'Signal'] = -1

df = df.drop(df.index[i+1:])  # i行目以降の行を削除
```
バックテストが正しく実行されているか確認する
移動平均線からの乖離率が+2以上又は-2以下でデータを抽出
```perl:標準偏差 +2以上 or -2以下
filtered_data = df[(df['Deviation'] >= 2) | (df['Deviation'] <= -2)][['Deviation','Signal','Position']]
filtered_data.tail(10)
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/00691e0d-decc-cddb-78c5-8337858e3192.png)  
乖離率が -2以下の時に買いシグナル、買いポジション  
乖離率が -2以下だが買いポジションを持っているのでシグナルが出ていない  
乖離率が +2以上なので売りシグナル、ポジション解除  
乖離率が +2以上なので売りシグナル、売りポジション  
正しく動作している  

## リターンの算出
```perl:初期資本 100万年、1ポジション = 100株
    capital = 1000000
    position_size = 100
```
```perl:株数変化列、株式資産、キャッシュ残高、総資産、リターン、累積リターンの算出
df['Shares'] = df['Position'] - df['Position'].shift()  # 株数変化列を作成
df['Shares'].fillna(0, inplace=True)  # 株数変化列の最初の値を0に設定

df['Portfolio'] = df['Adj Close'] * df['Position'] * position_size  # ポートフォリオ価値列を作成
df['Portfolio'].iloc[0] = 0  # 最初の日のポートフォリオ価値を0に設定

df['Cash'] = capital - (df['Shares'] * df['Adj Close'] * position_size).cumsum()  # キャッシュ残高列を作成
df['Value'] = df['Cash'] + df['Portfolio']  # 総資産列を作成

df['Returns'] = df['Value'].pct_change()  # リターン列を作成
df['Cumulative Returns'] = (1 + df['Returns']).cumprod()  # 累積リターン列を作成
```
:::note info  
Shares = 株数変化列  
Portfolio = 株式資産  
Cash = キャッシュ残高  
Value = 総資産  
Returns = リターン  
Cumulative Returns = 累積リターン  
:::  

結果を確認してみる  
```perl:バックテストの結果を表示
    print('Cumulative Returns:', df['Returns'].sum())
    print('Return:', df['Returns'].sum() * capital)  # リターンの計算
```
Cumulative Returns: -0.26667785862970095  
Return: -266677.85862970096  
視覚的に確認する  

```perl:プロットの設定
    plt.plot(df[['Returns','Cumulative Returns']])
    plt.legend(df[['Returns','Cumulative Returns']])
    plt.show()
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3471374/e8b777ce-9176-d473-e0ae-d9c0128b0bd9.png)  
2006年から2008年にかけて上昇、  
上記以外の期間は損失を出している  

# 結論
**－26万6000円**  
ボリンジャーバンドだけだと売買シグナルとしては使えない。  

**最後までお読みいただき、ありがとうございました。**  

:::note warn  
※条件を変えることで結果は変わります。  
    1. 銘柄を変える  
    2. ポジションを複数持つ  
    3. 初期資本を多くする  
    4. ロット数の調整  
    5. 損切の制定 etc...  
:::  
