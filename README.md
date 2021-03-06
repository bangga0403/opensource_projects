# **서울과학기술대학교 오픈소스프로그래밍**

## **프로젝트 개요**
LSTM(Long Short-Term Memory)를 이용하여 이더리움의 시간별 OHLCV(시가,최고가,최저가,종가,매매량)을 바탕으로 회귀분석(regression)을 통하여 이더리움의 가격을 예측하는 프로젝트.
  
## **요구 패키지**
* ccxt
* torch
* matplotlib
* sklearn
* numpy

## 들어가기 앞서
이 프로젝트는 서울과기대 컴퓨터공학부 오픈소스-프로그래밍 기간 프로젝트 제출용으로 만들어졌습니다. 학부생 수준으로 작성되었으며, 대부분의 코드는 수업시간 강의자료에 기반하였습니다.
때문에 정확성이 매우 떨어지며, 혹여나 이를 실제 코인 투자에 활용할 시 참고만 하시기 바랍니다.

## 사용법
상단 repository의 ETH_LSTM.py,binance_data_load.py,data폴더를 모두 다운받고, ETH_LSTM.py를 IDE를 이용해 실행합니다.
  
바이낸스 API를 생성하는 방법은 [파이썬을 이용한 비트코인 자동매매 (개정판)](https://wikidocs.net/120385)를 참조하세요.
API 키가 없어도 실행은 가능합니다. (다만, data폴더의 csv파일들을 업데이트 할 수 없습니다.)

## 기본구조
ETH_LSTM.py 를 실행시 코인 거래소인 바이낸스에서 자동으로 이더리움의 OHLCV 데이터를 가져와서 데이터셋을 업데이트합니다. 그 후 데이터셋에서 필요한 데이터를 추출하여
LSTM 기반 딥러닝을 진행한 후, 이더리움의 다음날 가격을 예측합니다. (MAE 오차도 같이 제공합니다.)
  
이때 ETH_LSTM.py의 t_frame를 적절히 변경하면 다음날이 아닌 12시간 뒤, 1시간 뒤 가격을 예측할 수 있습니다.
또한 LSTM의 layer나 epoch 수를 조정할 수도 있습니다. 관련 변수들은 후술하겠습니다.

# 파일별 설명
## binance_data_load.py
binance API를 이용하여 이더리움의 OHLCV 데이터를 가져와 csv파일로 저장하거나, 데이터를 가져오는 클래스인 binance_data 가 존재하는 .py파일입니다.
```python
import binance_data_load as BDL

bdl = BDL.binance_data(api_key,secret,symbol,t_frame) # binance_data 객체 생성
bdl.updating_coin_csv() # csv 파일 업데이트 함수

coindata = bdl.Load_Coin_Data() # ohlcv 데이터 로드 함수
coindata.data # ohlcv 데이터
coindata.targets # 미래 시가 종가 데이터
```


## ETH_LSTM.py
LSTM를 이용하여 딥러닝을 하는 .py파일입니다.

* binance 관련 변수
```python
# 바이낸스의 API 키 입력(없을시 '', 대신 최신 데이터 사용불가)
api_key = ''

# 바이낸스의 시크릿 키 입력(없을시 '', 대신 최신 데이터 사용불가)
secret = ''

# OHLCV의 시간 간격 (ex. '15m' -> 10시 15분 데이터, 10시 30분 데이터...)
t_frame = '1d' # '1m','15m','30m','1h','3h','1d'

# 가져올 코인 '코인/단위 임. 
symbol = 'ETH/USDT'
```
* LSTM 관련 변수
LSTM 관련 변수들로 적절히 변경 가능
```python
EPOCH_MAX = 50 # epoch 횟수
EPOCH_LOG = 1 # epoch 횟수별 표시 간격
OPTIMIZER_PARAM = {'lr': 0.01} # learning rate
SCHEDULER_PARAM = {'step_size': 5, 'gamma': 0.5} # scheduler param
USE_CUDA = torch.cuda.is_available() # CUDA 사용유무
SAVE_MODEL = f'./data/ETH_LSTM._{t_frame}pt' # Make empty('') if you don't want save the model
RANDOM_SEED = 777 # 랜덤시드 고정
DATA_LOADER_PARAM = {'batch_size': 50, 'shuffle': False} # data loader param
```

* LSTM layer
```python
class ETH_LSTM(nn.Module):
    def __init__(self, input_size, output_size):
        super(ETH_LSTM, self).__init__()
        self.LSTM1 = torch.nn.LSTM(input_size, 128)
        self.LSTM2 = torch.nn.LSTM(128,128)
        self.fc = torch.nn.Linear(128, output_size)

    def forward(self, x):
        output, hidden = self.LSTM1(x)
        output, hidden = self.LSTM2(output)
        x = self.fc(output) # Use output of the last sequence
        return x
```
LSTM layer는 기본적으로 2개의 Multi-layer를 이용하였습니다. pytorch를 이용해 레이어를 수정할 수 있습니다.

```python
# 1.1 normalizeData
    data_normalizer = preprocessing.MinMaxScaler()
    target_normalizer = preprocessing.MinMaxScaler()
    coin_data_normalized = data_normalizer.fit_transform(coin_data)
    coin_targets_normalized = target_normalizer.fit_transform(coin_targets)
    #test_data_normalized = data_normalizer.fit_transform(test_data)
```
dataset를 skilearn의 MinMaxScalar()를 이용하여 정규화 하였습니다.

## 출력결과
### t_frame = '1d' (즉, 하루를 기준으로)
  
![Tr_loss](./img/TrLoss.png)
![graph](./img/graph.png)

```bash
MAE : 85.14462921226303
2021-12-22 09:00:00 기준:
예상 시작가: 4100.94140625, 예상 종가: 4101.60107421875
```

## 발전방향
LSTM RNN이 아직 부실합니다. overfitting 되었을 가능성이 높습니다. dropout layer등을 추가하여 오차를 줄여야 합니다.
데이터셋이 과도하게 사용됩니다. 이 역시 overfitting 되었을 가능성이 높습니다. 이 프로젝트상에서는 모든 시계열데이터를
사용하고 있는데, 이더리움은 2020년도와 2021년도의 가격의 차이가 큰 만큼 과거데이터가 미치는 영향이 작을수 있습니다.
따라서 기간을 정하여 데이터셋을 분할하여 벡터를 형성하여야 합니다.(batch_size와는 다른 문제)

## 참고자료

### binance_data_load.py 관련
* [파이썬을 이용한 비트코인 자동매매 (개정판)](https://wikidocs.net/120385)
### ## ETH_LSTM.py 관련
* 오픈소스프로그래밍 강의자료 (DL.pdf) [관련 Github](https://github.com/mint-lab/dl_tutorial)
* [위대한 개스피 채널](https://www.youtube.com/watch?v=9haME49Rx_0)[관련 Colab](https://colab.research.google.com/drive/149QFMKdMu9iSQoD6MaQ7ON5sncHzdH3x?usp=sharing#scrollTo=asdN_HvQfTpw)

