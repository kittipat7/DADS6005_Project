# Final group project of DADS6005
Data Streaming and Realtime Analytics, วิทยาศาสตรมหาบัณฑิตสถิติประยุกต์ สถาบันบัณฑิตพัฒนบริหารศาสตร์

## สมาชิกในกลุ่ม
6410412002 กิตติภัทร ภัทรจริยา
6410412004 ชนธัญญา ยศบุตร
6410412010 ศรัณย์ ดิษเจริญ








# DADS6005_Project
# group member
6410412002 Mr. Kittipat Pattarajariya

6410412004 Miss Chonthanya Yosbuth

6410412010 Mr. Saran Ditjarern

# Video Presentation
https://drive.google.com/file/d/14l7nSjE6jKbPkD3AJ1FIjlsiqqd2iKq2/view?usp=sharing

# design diagram
![6005_diagram](https://user-images.githubusercontent.com/97491541/212450021-c0d95cd5-5574-463a-b621-b43be64995f4.jpg)

# 1. consumer

```
%%capture
!pip install confluent_kafka
!pip install plotly
!pip install chart_studio
!pip install skforecast
```
```
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
# 2. producer1
```
%%capture
!pip install confluent_kafka
!pip install cryptocompare
```
```
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
# 3. producer2
```
#coindesk
%%capture
!pip install confluent_kafka
#!pip install -U coindesk
```
```
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
# Api
https://min-api.cryptocompare.com/data/price

https://api.coindesk.com/v1/bpi/currentprice.json
