# Stock Analyzer
- 실시간으로 각 주식시장별 종목의 시세 조회 및 분석
- 누적된 데이터와 머신러닝 모델을 기반으로 주가 예측.  

위의 두 가지 기능을 중점으로 하는 웹 사이트를 목표로 합니다.

## 사용될 스택
- Django & DRF (Django Rest Framework)  
> 백엔드서버로써 사용합니다.  

- React.js & material ui
> 프론트엔드로써 사용합니다.  
 
- PyKrx
> 주가 정보 수집 모듈로써 사용됩니다.

- MongoDB  
> MongoDB를 채택한 이유는 스키마의 제약을 덜 받기 위함입니다. 

- Sklearn (Regression)  
> 주가 데이터를 회귀 분석을 통해 예측.  
Linear Regression > Ridge Regression > Lasso Regression  
전체적인 서비스가 구현이 되면 다른 모델을 도입하며 정확도를 높여나갈 예정입니다. 주가에 영향으 주는 독립변수(feature)들을 찾아야 합니다.  

- Matplotlib  
> 데이터 시각화 모듈.

## Case Rule  
클래스명 : Upper Camel case  
메소드, 속성 : Snake case

## 모듈 관리
모듈 관리는 requirements.txt를 통해 이루어진다.  
```
# 패키지 목록과 버전 텍스트로 저장하기
pip freeze > requirements.txt

# 텍스트 파일에 있는 패키지를 설치하기
pip install -r requirements.txt
```

## 계획 및 진행과정  

### 1. 백엔드서버 구축  
[>> detail..](./markdowns/01_backend.md)  

### 2. MongoDB 연동  
[>> detail..](./markdowns/02_db_connect.md)  

### 3. PyKrx를 통한 데이터 수집  
[>> detail..](./markdowns/03_db_collect.md)  

### 4. 프론트엔드 뷰 구축 (React.js)   
[>> detail..](./markdowns/04_frontend.md)  

### 5. Regression 적용  
참고 : https://steemit.com/kr/@phuzion7/volume  
pandas_datareader의 get_data_yahoo()를 통해 각 주식종목별 정보를 데이터 프레임 형태로 가져왔다. 그리고 여기서 volume의 의미는 다음 링크를 참고했다. 간략하게 설명하자면, volume은 일정기간의 거래량을 의미한다. 일반적으로 주가가 상승할 때에는 거래량이 증가한다고 한다. (거래를 시도하는 세력들에 의해) 크게 상승 볼륨, 하락 볼륨, U볼륨, 돔볼륨, 랜덤볼륨으로 나뉜다. 이중 볼륨의 상승과 하락은 선형회귀를 통해 파악할 수 있다.

### 6. 데이터 시각화  

### 7. 다른 머신러닝 모델 적용 및 테스트
