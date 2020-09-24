

```python
import os,sys
from configparser import ConfigParser
#os.chdir(os.path.dirname(os.path.abspath(__file__)))

def read_rkTrader_config(section):
    filename='../rkTraderConfig.ini'
    parser = ConfigParser()
    parser.read(filename)
    
     # get section, default to mysql
    paths = {}
    if parser.has_section(section):
        items = parser.items(section)
        for item in items:
            paths[item[0]] = item[1]
    else:
        raise Exception('{0} not found in the {1} file'.format(section, filename))
        
    return paths
```


```python
rkTrader_config = read_rkTrader_config('paths')
print(rkTrader_config['rktraderroot'])
os.chdir(rkTrader_config['rktraderroot'])
sys.path.append(rkTrader_config['rktraderroot'])
```

    C:\\Users\\ramkr\\Work\\GitProjects\\rkTrader
    


```python
import pandas as pd
import numpy as np
from datetime import datetime
import requests
import urllib

from rkAlerts import rkTrader_sendAlert
```


```python
def sendMessage(msg):
    msg = msg.replace("&","%26")
    rkTraderExperimentalTradesURL = "https://api.telegram.org/bot1126763246:AAGimQP3bDH1c7YJZ6odDJQeUehKxw9_hRQ/sendMessage?chat_id=-1001221021524&text=%s" % (msg,)
    r = requests.get(rkTraderExperimentalTradesURL)
```


```python
from dbConn import createConnection
conn = createConnection()
cursor = conn.cursor()
```

    Connecting to MySQL database...
    Connection not closed. Close it later
    


```python
trades = pd.read_sql("select * from bobdtrades /*where entrydate > '2020-08-18'*/ order by entrydate", con=conn)
```


```python
#len(trades)
```


```python
#trades.head(5)
```


```python
trades.loc[(trades['signal'] == "Short", "PnL")] = trades['entryprice'] - trades['exitprice']
trades.loc[(trades['signal'] == "Long", "PnL")] = trades['exitprice'] - trades['entryprice']

#trades
```


```python
trades['PnLPerc'] = trades['PnL'] / trades['entryprice'] * 100
```


```python
# Maximum Profit % from observed trades
round(trades['PnLPerc'].max(),2)
```




    34.42




```python
# Max loss % from observed trades
round(trades['PnLPerc'].min(),2)
```




    -24.71




```python
winRate = len(trades[trades['PnL'] > 0]) / len(trades)
message = "Current Win Rate of this model is "+str(round(winRate*100,2))+"%"
print(message)
#sendMessage(message)
```

    Current Win Rate of this model is 32.51%
    


```python
# Simulation configurations
initialCapital = 500000 # Starting capital
iterations = 1000 # Number ot trades per trial
trials = 101 # Number of individual simulations
riskPerTrade = 0.02 # 2% risk per trade
```


```python
import random
#random.randint(0,len(trades)-1)
```


```python
# Simulations 

results = pd.DataFrame(columns=['Trial','Outcome'])
indresults = []
finalCapital = 0

def simulateTrades(initialCapital,iterations,indresults):
    #indresults = []
    finalCapital = initialCapital
    #print(finalCapital)
    iterationCount = 0
    
    while iterationCount < iterations-1:
        currentTrade = trades.iloc[[random.randint(0,len(trades)-1)]]
        if currentTrade['signal'][currentTrade.index[-1]] == "Short":
            continue
            Risk = currentTrade['entryprice'][currentTrade.index[-1]] - currentTrade['firsttarget'][currentTrade.index[-1]]
        if currentTrade['signal'][currentTrade.index[-1]] == "Long":
            Risk = currentTrade['firsttarget'][currentTrade.index[-1]] - currentTrade['entryprice'][currentTrade.index[-1]]
            #continue
        if finalCapital < 10:
            finalCapital = 0
            iterationCount += 1
            continue
        positionSize = round(finalCapital*riskPerTrade / Risk)
        if positionSize < 0:
            iterationCount += 1
            continue
        tradePnL = 0
        if currentTrade['firsttargethit'][currentTrade.index[-1]] == 1:
            tradePnL = tradePnL + positionSize/2 * (currentTrade['firsttarget'][currentTrade.index[-1]] - currentTrade['entryprice'][currentTrade.index[-1]])
            positionSize = positionSize/2
        if currentTrade['secondtargethit'][currentTrade.index[-1]] == 1:
            tradePnL = tradePnL + positionSize/2 * (currentTrade['secondtarget'][currentTrade.index[-1]] - currentTrade['entryprice'][currentTrade.index[-1]])
            positionSize = positionSize/2
        tradePnL = tradePnL + positionSize * currentTrade['PnL'][currentTrade.index[-1]]
        finalCapital = finalCapital + tradePnL
        indresults.append(finalCapital)
        tradeMsg = "%s - %s with quantity of %s. Trade PnL = %s. Final Capital = %s" % (currentTrade['symbol'][currentTrade.index[-1]],currentTrade['signal'][currentTrade.index[-1]],positionSize,tradePnL,finalCapital)             
        #print(tradeMsg)
        finalTrade = currentTrade
        iterationCount += 1
    return finalCapital

trialCount = 0
while trialCount < trials-1:
    print(trialCount)
    finalCapital = simulateTrades(initialCapital,iterations,indresults)
    #print(finalCapital)
    results = results.append({'Trial':trialCount,'Outcome':finalCapital}, ignore_index=True)
    trialCount += 1
```

    0
    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    26
    27
    28
    29
    30
    31
    32
    33
    34
    35
    36
    37
    38
    39
    40
    41
    42
    43
    44
    45
    46
    47
    48
    49
    50
    51
    52
    53
    54
    55
    56
    57
    58
    59
    60
    61
    62
    63
    64
    65
    66
    67
    68
    69
    70
    71
    72
    73
    74
    75
    76
    77
    78
    79
    80
    81
    82
    83
    84
    85
    86
    87
    88
    89
    90
    91
    92
    93
    94
    95
    96
    97
    98
    99
    


```python
round(finalCapital,2)
```




    86897.89




```python
#results
```


```python
#results
```


```python
import matplotlib.pyplot as plt 
%matplotlib inline
#results = pd.Series(results)
#results.plot() 
plt.scatter(results['Trial'],results['Outcome'])
plt.show()
```


![png](output_19_0.png)



```python
# Number of times simulations ended with a profit from initial capital
len(results[results['Outcome']>initialCapital])
```




    15




```python
# Maximum profit ended up with in trials
round(results['Outcome'].max())
```




    1067347.0




```python
cursor.close()
conn.commit()
conn.close()
```


```python
#indresults.max()
```


```python
import matplotlib.pyplot as plt 
%matplotlib inline
indresults = pd.Series(indresults)
indresults.plot() 
#plt.scatter(results['Trial'],results['Outcome'])
plt.show()
```


![png](output_24_0.png)



```python
#import matplotlib.pyplot as plt

#x = [5,7,8,7,2,17,2,9,4,11,12,9,6]
#y = [99,86,87,88,111,86,103,87,94,78,77,85,86]

#plt.scatter(y)
#plt.show()
```
