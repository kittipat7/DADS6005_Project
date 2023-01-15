# Final group project วิชา DADS6005
Data Streaming and Realtime Analytics, วิทยาศาสตรมหาบัณฑิตสถิติประยุกต์ สถาบันบัณฑิตพัฒนบริหารศาสตร์

## สมาชิกในกลุ่ม
6410412002 กิตติภัทร ภัทรจริยา  
6410412004 ชนธัญญา ยศบุตร  
6410412010 ศรัณย์ ดิษเจริญ  

## วัตถุประสงค์  
เปรียบเทียบ Variance และ Mean Square Error (MSE) ของเหรียญคริปโตฯ ระหว่าง Ethereum และ Bitcoin  
เพื่อเปรียบเทียบ Variance และ Mean Square Error (MSE) มีความสัมพันธ์ในทิศทางเดียวกันหรือไม่  
โดยใช้ Libary skforecast  

##### Libary skforecast  
![image](https://user-images.githubusercontent.com/97492504/212503469-37995f25-9c68-44fb-82e7-4f6dcd53e7a0.png)  
สร้าง Model สำหรับการเปรียบเทียบ ซึ่ง Libary skforecast จะเปลี่ยนแปลง scikit-learn regressors กลายเป็น multi-step forecasters เพื่อง่ายต่อการสร้าง Model Real-Time โดยมี forecaster auto regression ที่เลือกมาใช้ก็คือ Random Rorest  

### Video Presentation  
https://drive.google.com/file/d/14l7nSjE6jKbPkD3AJ1FIjlsiqqd2iKq2/view?usp=sharing  

### Process Flow Diagram  
![6005_diagram](https://user-images.githubusercontent.com/97491541/212450021-c0d95cd5-5574-463a-b621-b43be64995f4.jpg)

#### 1. Producer: ส่งราคาเหรียญคริปโตฯ Ethereum และ Bitcoin ไป topic
- `Producer 1` ส่งราคาเหรียญคริปโตฯ Ethereum CryptoCompare API ไป topic 1
- `Producer 2` ส่งราคาเหรียญคริปโตฯ Bitcoin CoinDesk API ไป topic 2

##### 2. Consumer: ดึงราคาเหรียญคริปโตฯ จาก topic 1 และ topic 2
- `Consumer 1` ดึงราคาเหรียญคริปโตฯ Ethereum จาก topic 1
- `Consumer 2` ดึงราคาเหรียญคริปโตฯ Bitcoin จาก topic 2

#### 3. เปรียบเทียบ Variance และ Mean Square Error (MSE) ของเหรียญคริปโตฯ
- `เปรียบเทียบ Variance` และ `สร้างกราฟ Visualization` ระหว่าง Ethereum และ Bitcoin  
- `เปรียบเทียบ Mean Square Error (MSE)` และ `สร้างกราฟ Visualization` ระหว่าง Ethereum และ Bitcoin  

### Coding  

`Producer 1`:  
install confluent_kafka และ install cryptocompare สำหรับการส่งราคาเหรียญคริปโตฯ Ethereum CryptoCompare  
```python
%%capture
!pip install confluent_kafka
!pip install cryptocompare
```

- ส่งราคาเหรียญคริปโตฯ Ethereum CryptoCompare API ไป topic 1 ทุกๆ 1 นาที  
- API: https://min-api.cryptocompare.com/data/price  
```python
from confluent_kafka import Producer
import requests
import json
import time

# Set up the Kafka producer
p = Producer({'bootstrap.servers': 'ec2-13-229-46-113.ap-southeast-1.compute.amazonaws.com:9092'})

# Set the CryptoCompare API endpoint and any necessary headers or parameters
api_endpoint = 'https://min-api.cryptocompare.com/data/price'
params = {'fsym': 'ETH', 'tsyms': 'USD'}

# Retrieve data from the CryptoCompare API in a loop
while True:
    # Make a request to the CryptoCompare API
    response = requests.get(api_endpoint, params=params)
    data = response.json()

    # Convert the data to a string and produce it to Kafka
    data_str = json.dumps(data)
    print(data_str)
    p.produce('eth1', data_str.encode('utf-8'))
    p.flush()
    time.sleep(60)
```

`Producer 2`:  
install confluent_kafka และ install coindesk สำหรับการส่งราคาเหรียญคริปโตฯ Bitcoin CoinDesk 
```python
#coindesk
%%capture
!pip install confluent_kafka
#!pip install -U coindesk
```

- ส่งราคาเหรียญคริปโตฯ Bitcoin CoinDesk API ไป topic 2 ทุกๆ 1 นาที  
- https://api.coindesk.com/v1/bpi/currentprice.json  
```python
from confluent_kafka import Producer
import requests
import json
import time

# Set up the Kafka producer
p = Producer({'bootstrap.servers': 'ec2-13-229-46-113.ap-southeast-1.compute.amazonaws.com:9092'})

# Set the CryptoCompare API endpoint and any necessary headers or parameters
api_endpoint = "https://api.coindesk.com/v1/bpi/currentprice.json"

# Retrieve data from the CryptoCompare API in a loop
while True:
    # Make a request to the CryptoCompare API
    response = requests.get(api_endpoint)
    data = response.json()

    # Convert the data to a string and produce it to Kafka
    data_str = json.dumps(data)
    print(data_str)
    p.produce('btc1', data_str.encode('utf-8'))
    p.flush()
    time.sleep(60)
     
```

`consumer`:  
- install confluent_kafka และ install skforecast สำหรับเตรียมนำราคาเหรียญคริปโตฯ Ethereum และ Bitcoin จาก topic 1 และ topic 2 เพื่อเปรียบเทียบ  
- install plotly และ install chart_studio สำหรับการสร้างกราฟ Visualization เพื่อเปรียบเทียบ  
```python
%%capture
!pip install confluent_kafka
!pip install plotly
!pip install chart_studio
!pip install skforecast
```

- ดึงราคาเหรียญคริปโตฯ Ethereum และ Bitcoin จาก topic 1 และ topic 2 ทุกๆ 1 นาที และเก็บเป็น List เพื่อเปรียบเทียบ Variance และ Mean Square Error (MSE) ทุกๆ 1 นาที  
> ใช้ Random Forest สำหรับการสร้าง Model Autoregressive forecasters จาก skforecast และเก็บ Variance และ Mean Square Error (MSE) ในทุกๆ 1 นาที เป็น List เพื่อนำมาสร้างเป็น Dataframe  

> Rescale Dataframe ที่ได้หลังจากการสร้าง Model เนื่องจาก Model ที่สร้างเป็น Real-Time จึงจำเป็นต้อง Rescale หลังจากสร้าง Model เพื่อไม่ให้ส่งผลกระทบถึงค่า z-scroe ที่ได้ เพราะราคาจริงของเหรียญคริปโตฯ ระหว่าง Ethereum และ Bitcoin มีความแตกต่างกันสูง  
- สร้างกราฟ Visualization ของ Variance และ Mean Square Error (MSE) เพื่อเปรียบเทียบว่ามีความสัมพันธ์ในทิศทางเดียวกันหรือไม่  
> กราฟ Visualization ของ Variance และ Mean Square Error (MSE) จะเปลี่ยนแปลงทุกๆ 1 นาที  
  
<details>
<summary>Detail coding consumer</summary>

```python
from confluent_kafka import Consumer, KafkaError
import json
import time
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from IPython.display import display, clear_output
from sklearn.linear_model import LinearRegression
from sklearn.linear_model import Lasso
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline
from skforecast.ForecasterAutoreg import ForecasterAutoreg
from skforecast.ForecasterAutoregCustom import ForecasterAutoregCustom
from skforecast.ForecasterAutoregDirect import ForecasterAutoregDirect
from skforecast.model_selection import grid_search_forecaster
from skforecast.model_selection import backtesting_forecaster
from skforecast.utils import save_forecaster
from skforecast.utils import load_forecaster
from sklearn.preprocessing import MinMaxScaler


# Set up the Kafka consumer for topic 1
c1 = Consumer({
    'bootstrap.servers': 'ec2-13-229-46-113.ap-southeast-1.compute.amazonaws.com:9092',
    'group.id': 'my-consumer-group',
    'auto.offset.reset': 'latest'
})
c1.subscribe(['eth1'])
# Set up the Kafka consumer for topic 2
c2 = Consumer({
    'bootstrap.servers': 'ec2-13-229-46-113.ap-southeast-1.compute.amazonaws.com:9092',
    'group.id': 'mygroup',
    'auto.offset.reset': 'latest'
})
c2.subscribe(['btc1'])

btc=[]
eth=[]
btc_var=[]
eth_var=[]
i=1
i2=1
x_count=[]
x_count2=[]
err_sqr_btc=[]
err_sqr_eth=[]
mse_btc=[]
mse_eth=[]

# Consume messages from Kafka and print them to the console
while True:
    msg1 = c1.poll(1.0)

    if msg1 is None:
        continue
    if msg1.error():
        print("Consumer error: {}".format(msg1.error()))
        continue
    msg2 = c2.poll(1.0)

    if msg2 is None:
        continue
    if msg2.error():
        print("Consumer error: {}".format(msg2.error()))
        continue
    

    # Print the message from topic 1
    a=msg1.value().decode('utf-8')
    b=json.loads(a)
    eth.append(b["USD"])
    print("ETH :",b["USD"])
    # Print the message from topic 2
    c=msg2.value().decode('utf-8')
    d=json.loads(c)
    bpi = d['bpi']
    bpi_USD=bpi["USD"]

    btc.append(bpi_USD["rate_float"])
    print("BTC :",bpi_USD["rate_float"])
    btc_var.append(np.var(btc))
    eth_var.append(np.var(eth))
    #counter
    x_count.append(i)

    #print variance and create data frame of varince
    datadict = {'period':x_count,'BTC': btc_var, 'ETH': eth_var}
    df = pd.DataFrame(datadict)
    if len(btc)<2:
      df.plot(x="period", y=["BTC", "ETH"])
    #model
    if len(btc)>=2:
      datadict2 = {'period':x_count,'BTC': btc, 'ETH': eth}
      df2= pd.DataFrame(datadict2)

    #BTC_model
      forecaster = ForecasterAutoreg(
                regressor = RandomForestRegressor(random_state=123),
                lags      = len(btc)-1
             )

      forecaster.fit(y=df2["BTC"])
      from multiprocessing.sharedctypes import Value
      steps = 1
      predictions = forecaster.predict(steps=steps)
      btc_pred = predictions.item()
    #ETH_model
      forecaster2 = ForecasterAutoreg(
                regressor = RandomForestRegressor(random_state=123),
                lags      = len(eth)-1
             )

      forecaster2.fit(y=df2["ETH"])
      from multiprocessing.sharedctypes import Value
      steps = 1
      predictions2 = forecaster2.predict(steps=steps)
      eth_pred = predictions2.item()
      
      err_sqr_btc.append((predictions.item()-bpi_USD["rate_float"])**2)
      err_sqr_eth.append((predictions2.item()-b["USD"])**2)

      mse_btc.append(sum(err_sqr_btc)/len(err_sqr_btc))
      mse_eth.append(sum(err_sqr_eth)/len(err_sqr_eth))

      x_count2.append(i2)
      datadict_mse = {'period':x_count2,'BTC_MSE': mse_btc, 'ETH_MSE': mse_eth}
      df_mse= pd.DataFrame(datadict_mse)

      #Normalization BTC MSE and ETH MSE
      scaler2 = StandardScaler()   
      scaler2.fit(df_mse)
      df_scaled2 = scaler2.transform(df_mse)
      df_scaled2 = pd.DataFrame(df_scaled2, columns=df_mse.columns)
      df_scaled2['mse_period'] = x_count2
      
      #Normalization BTC Variance and ETH Variance
      scaler = StandardScaler()
      scaler.fit(df)
      df_scaled = scaler.transform(df)
      df_scaled = pd.DataFrame(df_scaled, columns=df.columns)
      df_scaled['var_period'] = x_count

      #Visualize
      fig, (ax1, ax2) = plt.subplots(nrows=1, ncols=2, figsize=(25, 5))
      plt.subplots_adjust(wspace=0.5)
      ax1.set_title('Compare Variance between BTC and ETH')
      ax2.set_title('Compare MSE between BTC and ETH')

      df_scaled.plot(x="var_period", y=["BTC", "ETH"],ax=ax1)
      df_scaled2.plot(x="mse_period", y=["BTC_MSE", "ETH_MSE"],ax=ax2)

      print("BTC_pred :",predictions.item())
      print("ETH_pred :",predictions2.item())
      print("BTC_var :",df_scaled.tail(1)['BTC'].item())
      print("ETH_var :",df_scaled.tail(1)['ETH'].item())
      print("mse_btc :",mse_btc[-1])
      print("mse_eth :",mse_eth[-1])
      print("mse_btc_std :",df_scaled2.tail(1)['BTC_MSE'].item())
      print("mse_eth_std :",df_scaled2.tail(1)['ETH_MSE'].item())
      i2+=1

    plt.show()
    clear_output(wait=True)
    i+=1
    time.sleep(60)

c.close()
```
</details>


## สรุปผลลัพธ์
ตัวอย่างของผลลัพธ์จาก `Producer 1` ใน 1 นาที
![image](https://user-images.githubusercontent.com/97492504/212519860-fbbc7ea4-b553-4229-b910-9783abb793a6.png)  
ตัวอย่างของผลลัพธ์จาก `Producer 2` ใน 1 นาที
![image](https://user-images.githubusercontent.com/97492504/212519870-620dc57f-0c73-49a6-8afd-2779fa47d258.png)  
ตัวอย่างของผลลัพธ์จาก `consumer` และ `กราฟ  Visualization` ใน 1 นาที
![image](https://user-images.githubusercontent.com/97492504/212519889-9d6d3253-ab2e-4080-a56d-c2b9d7921edd.png)

### สรุปจากผลลัพธ์ที่ได้รับจากการเก็บข้อมูล 100 ตัว   
กราฟ  Visualization ของ Variance ให้ค่า error แตกต่างกันมาก แต่  
กราฟ  Visualization ของ Mean Square Error (MSE) ให้ค่า error คล้ายคลึงกัน  
ซึ่งจะเห็นว่า Variance และ Mean Square Error (MSE) ไม่มีความสัมพันธ์กัน ซึ่งผลลัพธ์จากการเก็บข้อมูล 100 ตัวไม่สอดคล้องกับที่ควรจะเป็นแม้จะทำการ Rescale แล้วก็ตาม เนื่องจากราคาจริงของ Bitcoin สูงกว่า 
Ethereum อย่างมาก ทำให้โอกาสในการขึ้น-ลง ของ Bitcoin สูงกว่า Ethereum 


## Reference  
https://joaquinamatrodrigo.github.io/skforecast/0.4.3/index.html  
https://pypi.org/project/cryptocompare/  
https://pypi.org/project/coindesk/  
