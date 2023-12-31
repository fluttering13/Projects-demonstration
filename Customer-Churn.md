# 顧客流失率分析 Customer-Churn 
資料集來源取至Kaggle:Telco Customer Churn

https://www.kaggle.com/datasets/blastchar/telco-customer-churn/discussion

以下是我們之後會用到的包

numpy pandas 兩個好朋友

還有從sklearn拿出來的SVM 跟 一些拆data或是feature轉換的工具

後面還有一個沒有在這邊特別寫的beta encoding

https://www.kaggle.com/code/mmotoki/beta-target-encoding
```
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

from sklearn.model_selection import train_test_split 
from sklearn.preprocessing import LabelEncoder, MinMaxScaler

import pickle
from sklearn.svm import SVC
from sklearn.metrics import classification_report, confusion_matrix
```

# 資料前處理

先來看看表格屬性有什麼
```
columns=data_pd.columns
print(columns)
```

```
Index(['customerID', 'gender', 'SeniorCitizen', 'Partner', 'Dependents',
       'tenure', 'PhoneService', 'MultipleLines', 'InternetService',
       'OnlineSecurity', 'OnlineBackup', 'DeviceProtection', 'TechSupport',
       'StreamingTV', 'StreamingMovies', 'Contract', 'PaperlessBilling',
       'PaymentMethod', 'MonthlyCharges', 'TotalCharges', 'Churn'],
      dtype='object')
```

檢查看看有沒有缺失的檔案要padding
```
for name in columns:
    cond1=data_pd.isnull()[name]==True
    print(data_pd[cond1])
```

```
Index: []
Empty DataFrame
```
看來裡面的資料都是填好填滿，但是index的部分可能要填補

再來來粗略看一下有多少feature，再來決定等等要使用encoding的方式

```
print(data_pd.nunique())
for name in data_pd.columns:
    print(pd.unique(data_pd[name]))
```

```
customerID          7043
gender                 2
SeniorCitizen          2
Partner                2
Dependents             2
tenure                73
PhoneService           2
MultipleLines          3
InternetService        3
OnlineSecurity         3
OnlineBackup           3
DeviceProtection       3
TechSupport            3
StreamingTV            3
StreamingMovies        3
Contract               3
PaperlessBilling       2
PaymentMethod          4
MonthlyCharges      1585
TotalCharges        6531
Churn                  2
dtype: int64
```
```
['7590-VHVEG' '5575-GNVDE' '3668-QPYBK' ... '4801-JZAZL' '8361-LTMKD'
 '3186-AJIEK']
['Female' 'Male']
[0 1]
['Yes' 'No']
['No' 'Yes']
[ 1 34  2 45  8 22 10 28 62 13 16 58 49 25 69 52 71 21 12 30 47 72 17 27
  5 46 11 70 63 43 15 60 18 66  9  3 31 50 64 56  7 42 35 48 29 65 38 68
 32 55 37 36 41  6  4 33 67 23 57 61 14 20 53 40 59 24 44 19 54 51 26  0
 39]
['No' 'Yes']
['No phone service' 'No' 'Yes']
['DSL' 'Fiber optic' 'No']
['No' 'Yes' 'No internet service']
['Yes' 'No' 'No internet service']
['No' 'Yes' 'No internet service']
['No' 'Yes' 'No internet service']
['No' 'Yes' 'No internet service']
['No' 'Yes' 'No internet service']
['Month-to-month' 'One year' 'Two year']
['Yes' 'No']
['Electronic check' 'Mailed check' 'Bank transfer (automatic)'
 'Credit card (automatic)']
[29.85 56.95 53.85 ... 63.1  44.2  78.7 ]
['29.85' '1889.5' '108.15' ... '346.45' '306.6' '6844.5']
['No' 'Yes']
```

## 分布可視化 (feature-frequency-plot)

從以上可以得知，連續性的資料有這三項
```
data_pd_numberical=data_pd[['tenure','MonthlyCharges','TotalCharges']]
```
首先我們可以sub polt一下
```
features=data_pd_numberical.columns
for i in range(0, len(features)):
    plt.subplot(2, len(features)//2 + 1, i+1)
    plt.hist(data_pd_numberical[features[i]], bins=50)
    plt.xlabel(features[i])
plt.show()
```
![image](./Customer-Churn/pic/hist1.png)

1. 從Tenure可以看到有雙峰的情況，大多數人群要馬就是持有很久或是剛持有
，市場策略可以對這兩種族群進行處理

2. 從MonthlyCharges可見大多數人為低資費，並且從40開始到120結束有呈一個常態分佈
意味著收費策略可以對這兩種族群進行處理

3. TotalCharge 是一個均勻分布

## 回顧率分析與可視化 (histogram plot)
我們將雙變數或是多變數做直方圖
```
features = ['gender', 'SeniorCitizen', 'Partner', 'Dependents',
       'tenure', 'PhoneService', 'MultipleLines', 'InternetService',
       'OnlineSecurity', 'OnlineBackup', 'DeviceProtection', 'TechSupport',
       'StreamingTV', 'StreamingMovies', 'Contract', 'PaperlessBilling',
       'PaymentMethod', 'MonthlyCharges', 'TotalCharges']
for feature in features:
    sns.histplot(data = data_pd, x = feature, hue = 'Churn', multiple = 'dodge', bins=30)
    plt.title(f'Histogram of {feature} with Churn')
    plt.xlabel(feature)
    plt.ylabel('Frequency')
    plt.show()
```
直接說重點
1. 大部分的項目 不願意續簽的人>願意續簽 這也是我們想要理解的問題
2. 願不願意續約與性別無關
3. 不願意續約的群體以年輕人為大宗
4. 願意續約的群體以單身人群為多數，不願意續約的群體以有伴侶的人為多數
<div align=center><img src="./Customer-Churn/pic/bihis1.png" width="400px"/></div>
5. 願意續約的群體以有家人的為多數
6. 呈之前tenure提過的，一多數群體為新戶，從70開始突然出現多數退約，值得注意
<div align=center><img src="./Customer-Churn/pic/his_tenure.png" width="400px"/></div>
7. 無論是無續約，使用電話服務的群體為大宗。

8. 願不願意續約與有無使用多線服務可能無關
<div align=center><img src="./Customer-Churn/pic/his_int.png" width="400px"/></div>
9.  在願意續約的群體以光纖網路為大宗，在不願意續約的群體以DSL為大宗
意味著群體的轉化或許可以透過推廣服務進行

10. 願不願意續約與在線安全服務無關
    
11. 在願意續約的群體，以沒有網路備份為大宗
<div align=center><img src="./Customer-Churn/pic/his_ob.png" width="400px"/></div>

12. 有無定閱資安服務或是串流看起來與願不願意續約沒什麼關係
<div align=center><img src="./Customer-Churn/pic/his_life.png" width="400px"/></div>
13. 訂閱時間以第一年開始大量跳樓

14. 有無續約都以紙本服務為多數

15. 有續約的群體以電子支票為多數

16. 從資費為20附近為拒絕續約的最大宗
    
<div align=center><img src="./Customer-Churn/pic/hist_pay.png" width="400px"/></div>

## 特例: 資費小等於20
再來我們從資費小等於20為拒絕續約的最大宗為條件，在重新看一次分布去了解問題

我們可以懷疑也許費率是一個影響最大的因素

於是我們將資費變成一個控制變因，再從頭來看有沒有還可以把握什麼次要的因素

```
cond=data_pd['MonthlyCharges']<=20
new_df=data_pd[cond]
features = ['gender', 'SeniorCitizen', 'Partner', 'Dependents',
       'tenure', 'PhoneService', 'MultipleLines', 'InternetService',
       'OnlineSecurity', 'OnlineBackup', 'DeviceProtection', 'TechSupport',
       'StreamingTV', 'StreamingMovies', 'Contract', 'PaperlessBilling',
       'PaymentMethod', 'MonthlyCharges', 'TotalCharges']
for feature in features:
    sns.histplot(data = new_df, x = feature, hue = 'Churn', multiple = 'dodge', bins=30)
    plt.title(f'Histogram of {feature} with Churn')
    plt.xlabel(feature)
    plt.ylabel('Frequency')
    plt.show()
```
1. 從持占率裡面看更慘了
<div align=center><img src="./Customer-Churn/pic/his_s2ten.png" width="400px"/></div>
2. 從便宜資費的群體可見以mail checked為大宗
   
也許可以改善郵件推廣作為著手點出發

3. Monthly charges 呈線性增加
<div align=center><img src="./Customer-Churn/pic/his_s2mon.png" width="400px"/></div>   

4. 其他的觀察都與上一個條目看到的差不多
   


# 分類器任務SVM (SVM classification)
此處我們利用SVM來進行分類器任務

主要利用libSVM的包

首先先把不重要的ID踢掉
```
data_pd.drop(columns=["customerID"], inplace=True)
```

使用label encoder比較省事，把yes跟no那些轉成0跟1

再來使用train_test_split來拆訓練用跟測試用的data

最後用MinMaxScaler來做標準化
```
le = LabelEncoder()
for c in data_pd.columns:
    data_pd[c] = le.fit_transform(data_pd[c])

cols = ['gender', 'SeniorCitizen', 'Partner', 'Dependents', 'tenure',
       'PhoneService', 'MultipleLines', 'InternetService', 'OnlineSecurity',
       'OnlineBackup', 'DeviceProtection', 'TechSupport', 'StreamingTV',
       'StreamingMovies', 'Contract', 'PaperlessBilling', 'PaymentMethod',
       'MonthlyCharges', 'TotalCharges', ]
X_train, X_test, y_train, y_test = train_test_split(data_pd[cols], data_pd["Churn"], train_size=0.8, random_state=42)

mms = MinMaxScaler()
X_train=mms.fit_transform(X_train)
X_test=mms.fit_transform(X_test)
```
再來就是使用SVC package
```
svc = SVC()
svc.fit(X_train, y_train)
y_pred = svc.predict(X_test)
CM=confusion_matrix(y_test, y_pred)
```

## 元優化
接下來我們來進行一些自動化的ML訓練
使用optuna的包
```
def objective(trial):
    param_grid = {
        "C":trial.suggest_float("C",50,70),
        "degree":trial.suggest_int("degree",1,3),
        #'kernel': trial.suggest_categorical("kernel", ["poly", "rbf","sigmoid"])
        'kernel': trial.suggest_categorical("kernel", ["poly"])
    }
    svc = SVC(**param_grid)
    svc.fit(X_train, y_train)
    # y_pred = svc.predict(X_test)
    pickle.dump(svc, open(f_path+str(trial.number)+"_svc.m",'wb'))
    # CM=confusion_matrix(y_test, y_pred)
    # acc=(CM[0,0]+CM[1,1])/CM.sum()
    return svc.score(X_test, y_test)

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials = 100)
print('Number of finished trials:', len(study.trials))
print('Best trial parameters:', study.best_trial.params)
print('Best score:', study.best_value)

fig=optuna.visualization.matplotlib.plot_optimization_history(study)
plt.show()
fig2 = optuna.visualization.matplotlib.plot_param_importances(study)
plt.show()
```
跑了一百次這樣的任務，驗證率可以拿到大概82%

```
Best trial parameters: {'C': 60.882149968640974, 'degree': 1, 'kernel': 'poly'}
Best score: 0.8197303051809794
```

## BETA encoding
回顧一下，前面的feature encoding，其實做了一些不太好的地方

針對一些categorical feature的地方 例如說有 0,1,2 各自map到 0.3,0.6,0.9 

但categorical feature並沒有這個數值高低的關係

所以要透過一些後驗的方式去修正這個高低

所以我們可以使用feature encoding的方式來幫助提取特徵

這邊使用的是Beta encoding

使用一樣的setting 準確率直接爆衝到94.5%

再做一次
```
Best trial parameters: {'C': 79.26227943687404, 'degree': 3, 'kernel': 'poly'}
Best score: 0.9453513129879347
```

# 同場加映，因果推斷之干預

我們做完前面的多變數分析之後，大部分我們還是利用眼睛去看

但有時候我們會被correlation跟confounding factor所蒙蔽

每個月用戶的消費力會影響到我們要看的tenure 也許也會與partner，payment way有關 

我們可以先假設我們的因果關係是長成下面這張圖
<div align=center><img src="./Customer-Churn/pic/c1.png" width="400px"/></div>  

再來我們想作的事情是打斷這個鍵結，想要知道的是我們想看的tenure, partner......多會影響到最後的churn

<div align=center><img src="./Customer-Churn/pic/c2.png" width="400px"/></div> 

這邊我們作的方法是作intervention，不熟悉的可以看下面

https://github.com/fluttering13/Causal-models/blob/main/Thompson_paradox_and_intervention.md

```
def create_intervention_AB(data_number,data_dead_number):

    data_number=data_number.assign(total=data_number.sum(axis=1))
    data_dead_number=data_dead_number.assign(total=data_dead_number.sum(axis=1))

    indexes=data_number.index
    data_rates=data_dead_number/data_number

    data_rates=data_rates.rename(columns={'total':'correlation'})


    new_list=[]
    c_numbers=data_number.sum()
    for i in range(data_rates.shape[0]):
        tmp_sum=0
        for j in range(data_rates.shape[1]-1):
            E_y_given_t_c=data_rates.iloc[i,j]
            p_c=c_numbers.iloc[j]/c_numbers.iloc[-1]
            tmp_sum=tmp_sum+E_y_given_t_c*p_c
        new_list.append(tmp_sum)
    new_list=pd.DataFrame({'intervention':new_list},index=indexes)
    data_rates=pd.concat([data_rates,new_list],axis=1)

    return data_rates
```

跟上面一樣使用這個function input一個是實驗的總數，另外一個是實驗中要看的數值

以這個例子來說，就是分別做tenure、Partner，我們總共做三個實驗

對於連續變數，我們就用平均值來分組，

表格裡面的值代表條件機率

h_MC代表大於MC平均，l_MC代表小於MC平均

以tenure的index來說：0代表大於tenure平均 1則是小於

以partner的index來說：0代表單身 1則是有伴

以payment的index來說，0到3分別為：

['Electronic check' 'Mailed check' 'Bank transfer (automatic)' 'Credit card (automatic)']

```
MC_ave=data_pd_numberical['MonthlyCharges'].mean()

# data_pd_numberical['tenure']=pd.to_numeric(data_pd_numberical['tenure'])

def tenure_causal_test():
    ave=data_pd_numberical['tenure'].mean()

    cond_list=[]  
    cond_list.append((data_pd_numberical['tenure']>=ave) & (data_pd_numberical['MonthlyCharges']>=MC_ave))
    cond_list.append((data_pd_numberical['tenure']>=ave) & (data_pd_numberical['MonthlyCharges']<MC_ave))
    cond_list.append((data_pd_numberical['tenure']<ave) & (data_pd_numberical['MonthlyCharges']>=MC_ave))
    cond_list.append((data_pd_numberical['tenure']<ave) & (data_pd_numberical['MonthlyCharges']<MC_ave))

    cond_list2=[]  
    cond_list2.append((data_pd_numberical['tenure']>=ave) & (data_pd_numberical['MonthlyCharges']>=MC_ave) & (data_pd_numberical['Churn']==1))
    cond_list2.append((data_pd_numberical['tenure']>=ave) & (data_pd_numberical['MonthlyCharges']<MC_ave) & (data_pd_numberical['Churn']==1))
    cond_list2.append((data_pd_numberical['tenure']<ave) & (data_pd_numberical['MonthlyCharges']>=MC_ave) & (data_pd_numberical['Churn']==1))
    cond_list2.append((data_pd_numberical['tenure']<ave) & (data_pd_numberical['MonthlyCharges']<MC_ave) & (data_pd_numberical['Churn']==1))



    highMC_list=[]
    lowMC_list=[]
    highMC_list.append(data_pd_numberical[cond_list[0]]['customerID'].count())
    highMC_list.append(data_pd_numberical[cond_list[2]]['customerID'].count())
    lowMC_list.append(data_pd_numberical[cond_list[1]]['customerID'].count())
    lowMC_list.append(data_pd_numberical[cond_list[3]]['customerID'].count())
    highMC_list2=[]
    lowMC_list2=[]
    highMC_list2.append(data_pd_numberical[cond_list2[0]]['customerID'].count())
    highMC_list2.append(data_pd_numberical[cond_list2[2]]['customerID'].count())
    lowMC_list2.append(data_pd_numberical[cond_list2[1]]['customerID'].count())
    lowMC_list2.append(data_pd_numberical[cond_list2[3]]['customerID'].count())


    total_df=Df({'h_MC':highMC_list,'l_MC':lowMC_list})
    df=Df({'h_MC':highMC_list2,'l_MC':lowMC_list2})

    data_rates=create_intervention_AB(total_df,df)
    print(data_rates)



def partner_causal_test():
    cond_list=[]  
    cond_list.append((data_pd_numberical['Partner']==1) & (data_pd_numberical['MonthlyCharges']>=MC_ave))
    cond_list.append((data_pd_numberical['Partner']==1) & (data_pd_numberical['MonthlyCharges']<MC_ave))
    cond_list.append((data_pd_numberical['Partner']==0) & (data_pd_numberical['MonthlyCharges']>=MC_ave))
    cond_list.append((data_pd_numberical['Partner']==0) & (data_pd_numberical['MonthlyCharges']<MC_ave))

    cond_list2=[]  
    cond_list2.append((data_pd_numberical['Partner']==1) & (data_pd_numberical['MonthlyCharges']>=MC_ave)& (data_pd_numberical['Churn']==1))
    cond_list2.append((data_pd_numberical['Partner']==1) & (data_pd_numberical['MonthlyCharges']<MC_ave)& (data_pd_numberical['Churn']==1))
    cond_list2.append((data_pd_numberical['Partner']==0) & (data_pd_numberical['MonthlyCharges']>=MC_ave)& (data_pd_numberical['Churn']==1))
    cond_list2.append((data_pd_numberical['Partner']==0) & (data_pd_numberical['MonthlyCharges']<MC_ave)& (data_pd_numberical['Churn']==1))

    highMC_list=[]
    lowMC_list=[]
    highMC_list.append(data_pd_numberical[cond_list[0]]['customerID'].count())
    highMC_list.append(data_pd_numberical[cond_list[2]]['customerID'].count())
    lowMC_list.append(data_pd_numberical[cond_list[1]]['customerID'].count())
    lowMC_list.append(data_pd_numberical[cond_list[3]]['customerID'].count())
    highMC_list2=[]
    lowMC_list2=[]
    highMC_list2.append(data_pd_numberical[cond_list2[0]]['customerID'].count())
    highMC_list2.append(data_pd_numberical[cond_list2[2]]['customerID'].count())
    lowMC_list2.append(data_pd_numberical[cond_list2[1]]['customerID'].count())
    lowMC_list2.append(data_pd_numberical[cond_list2[3]]['customerID'].count())


    total_df=Df({'h_MC':highMC_list,'l_MC':lowMC_list})
    df=Df({'h_MC':highMC_list2,'l_MC':lowMC_list2})

    data_rates=create_intervention_AB(total_df,df)
    print(data_rates)


def payment_causal_test():
    cond_list=[]
    for i in range(4):  
        cond_list.append((data_pd_numberical['PaymentMethod']==i) & (data_pd_numberical['MonthlyCharges']>=MC_ave))
        cond_list.append((data_pd_numberical['PaymentMethod']==i) & (data_pd_numberical['MonthlyCharges']<MC_ave))
    cond_list2=[]
    for i in range(4):  
        cond_list2.append((data_pd_numberical['PaymentMethod']==i) & (data_pd_numberical['MonthlyCharges']>=MC_ave)& (data_pd_numberical['Churn']==1))
        cond_list2.append((data_pd_numberical['PaymentMethod']==i) & (data_pd_numberical['MonthlyCharges']<MC_ave)& (data_pd_numberical['Churn']==1))
    highMC_list=[]
    lowMC_list=[]
    highMC_list2=[]
    lowMC_list2=[]

    for i in range(0,8,2):
        highMC_list.append(data_pd_numberical[cond_list[i]]['customerID'].count())
        highMC_list2.append(data_pd_numberical[cond_list2[i]]['customerID'].count())
    for i in range(1,8,2):
        lowMC_list.append(data_pd_numberical[cond_list[i]]['customerID'].count())
        lowMC_list2.append(data_pd_numberical[cond_list2[i]]['customerID'].count())
    total_df=Df({'h_MC':highMC_list,'l_MC':lowMC_list})
    df=Df({'h_MC':highMC_list2,'l_MC':lowMC_list2})
    data_rates=create_intervention_AB(total_df,df)
    print(data_rates)

tenure_causal_test()
partner_causal_test()
payment_causal_test()
```

最後我們得到下面的結果，代表說其實跟我們看到correlation的結果差不多

數值間的順位關係沒有改變

這些結果並沒有被混淆因子影響太多

```
       h_MC      l_MC  correlation  intervention
0  0.169820  0.048882     0.125153      0.116245
1  0.539742  0.237846     0.386755      0.406004
       h_MC      l_MC  correlation  intervention
0  0.261084  0.101312     0.196649      0.190306
1  0.435816  0.214531     0.329580      0.337788
       h_MC      l_MC  correlation  intervention
0  0.498270  0.328051     0.452854      0.422864
1  0.299754  0.154357     0.191067      0.235344
2  0.220994  0.090767     0.167098      0.163305
3  0.192702  0.097674     0.152431      0.150606
```