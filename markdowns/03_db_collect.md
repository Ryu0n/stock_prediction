# 3-1 주식 종목 코드 수집
```python
class StockItemCodeCollector:
    def __init__(self):
        self.markets = ['KOSPI', 'KOSDAQ', 'KONEX', 'ALL']
        self.tickers = dict()
        for market in self.markets:
            self.tickers[market] = self.__collect_tickers(market)

    @staticmethod
    def __collect_tickers(market='KOSPI'):
        """
        stock.get_market_ticker_list()
        >> ['095570', '006840', '027410', '282330', '138930', ...]

        stock.get_market_ticker_name(ticker)
        >> SK하이닉스
        """
        tickers = dict()
        today = datetime.today().strftime("%Y%m%d")

        for ticker in stock.get_market_ticker_list(today, market):
            ticker_name = stock.get_market_ticker_name(ticker)
            tickers[ticker_name] = ticker
        return tickers

    def get_code(self, market, ticker_name):
        return self.tickers[market][ticker_name]
```
StockItemCodeCollector 인스턴스는 각 주식시장(KOSPI, KOSDAQ, KONEX, 전체)별로 딕셔너리에 각 종목의 이름과 코드를 보관하고 있다. get_code() 메소드를 통해 주식시장과 종목명을 입력하면 해당 종목 코드를 반환한다.  

# 3-2 주식 종목 데이터 수집
```python
class StockItemDataFrameCollector:
    def __init__(self, code):
        self.code = code
        self.today = datetime.now()
        self.today_str = self.today.strftime("%Y%m%d")

    def get_dataframe_from_previous(self, days=1):
        interval = timedelta(days)
        previous = self.today - interval
        previous_str = previous.strftime("%Y%m%d")
        dataframe = stock.get_market_ohlcv_by_date(previous_str, self.today_str, self.code)
        return dataframe
```
StockItemDataFrameCollector 인스턴스는 생성자에서 어떤 종목의 정보를 수집할지 정한다. get_dataframe_from_previous() 메소드를 통해 오늘날짜로부터 원하는 날 이전까지의 주가 정보를 데이터 프레임으로 반환한다.  

![image](https://user-images.githubusercontent.com/32003817/107649815-30507800-6cc1-11eb-9c41-57bfaef96eb8.png)
삼성전자의 주가를 일정한 주기(10초)로 수집하는 모습  

# 3-3 DRF를 활용한 데이터 삽입 (CREATE)
데이터 삽입을 위해 우선 django 모델의 포맷에 적합하도록 변환하는 과정을 진행했다.  
```python
class Converter:
    @classmethod
    def convert_to_json_for_item(cls, df: DataFrame, market: int, company: str):
        json_list = list()
        for index in df.index:
            # 각 날짜별 행을 딕셔너리로 가져오는 과정
            index_per_date: str = index.strftime('%Y-%m-%d')  # index: TimeStamp
            series_per_date: Series = df.loc[index_per_date]
            json_str_per_date: str = series_per_date.to_json(force_ascii=False)
            json_per_date: dict = json.loads(json_str_per_date)

            # 주식시장, 종목, 등록일 삽입
            json_per_date['stock_market'] = market
            json_per_date['stock_item_name'] = company
            json_per_date['reg_date'] = index_per_date

            # 한글로 된 키를 영문으로 변경
            json_per_date = cls.__convert_key_from_kor_to_eng_for_item(json_per_date)

            json_list.append(json_per_date)
        return json_list

    @staticmethod
    def __convert_key_from_kor_to_eng_for_item(json_per_date: dict):
        replacements = {'시가': 'open',
                        '종가': 'close',
                        '고가': 'high',
                        '저가': 'low',
                        '거래량': 'volume'}

        for key_kor, key_eng in replacements.items():
            json_per_date[key_eng] = json_per_date.pop(key_kor)

        return json_per_date
```  

간단히 설명하자면 PyKrx를 통해 가져온 데이터프레임을 requests 모듈의 인자로 보낼 수 있도록 딕셔너리로 변환하는 작업이다. 그리고 requests 모듈의 get 요청은 원할했지만 post요청을 하자 403 에러가 나타나는 것을 알 수 있었다. 오류내용은 다음과 같았다.  
```python
{
    "detail": "CSRF Failed: CSRF token missing or incorrect."
}
```  

결론부터 이야기 하면 이와 같은 오류가 나는 이유는 **로그인 세션**이 존재하지 않기 때문이다. 우리는 로그인 세션을 활성화하기 위한 토큰을 먼저 준비해야한다. (로그인을 위한 데이터라고 이해하면 쉽다.) 우선, 토큰을 통한 인증방식을 진행할 것이기 때문에 settings.py에서는 다음과 같이 설정해주어야 한다.  
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.TokenAuthentication',
    )
}
```  

다음으로는 로그인 세션을 생성해야한다.  
```python
class BaseReq:
    def __init__(self):
        self.url = 'http://127.0.0.1:8000/rest_api/'
        self.login_url = self.url + 'auth/login/'
        # 2021.02.14.hsk : 추후에 screts.json으로 통합
        self.id = ...
        self.password = ...
        self.data = None
        self.client, self.login_res = self._login(self.login_url, self.id, self.password)
        
    @staticmethod
    def _login(login_url, id, password):
        client = requests.session()
        client.get(login_url)
        csrf_token = client.cookies['csrftoken']
        login_data = {'Username': id, 'Password': password, 'csrfmiddlewaretoken': csrf_token}
        login_res = client.post(login_url, data=login_data)
        return client, login_res
    
    ...
```  

그리고 지금부터는 requests 모듈이 아닌 self.client 객체 (즉, 현재 활성화된 로그인 세션)을 통해서 요청을 해야한다.  
```python
class BaseCreate(BaseReq):
    def __init__(self):
        super().__init__()

    @csrf_exempt
    def send_post(self):
        for datum in self.data:
            res = self.client.post(self.url, json=datum)  # self.client(현재 활성화된 로그인 세션)를 사용한 요청
            res = self._ret(res)
            print(res, datum)
```  

![image](https://user-images.githubusercontent.com/32003817/107878707-cc73bc80-6f17-11eb-9a0d-1adba26d3ec9.png)  
![image](https://user-images.githubusercontent.com/32003817/107878749-05139600-6f18-11eb-93b9-bb83d640e97b.png)  
성공적으로 데이터가 삽입된 모습

* 참고  
CSRF는 무엇인가? CSRF는 Cross-site request forgery의 약자로 '사이트 간 요청 위조'를 의미한다. 사이트 간 스크립팅(XSS) 공격은 공격자가 웹 사이트에 악의적인 스크립트를 교묘하게 끼워넣어 사용자가 특정 웹사이트를 신용하는 점을 노린 것이라면, CSRF는 그와 반대로 특정 웹사이트가 사용자의 웹 브라우저를 신용하는 상태를 노린 것이다. 일단 사용자가 웹사이트에 로그인한 상태에서 사이트간 요청 위조 공격 코드가 삽입된 페이지를 열면, 공격 대상이 되는 웹사이트는 위조된 공격 명령이 믿을 수 있는 사용자로부터 발송된 것으로 판단하게 되어 공격에 노출된다. **CSRF 토큰이 존재하는 이유는 CSRF 공격을 방지하기 위해 고유한 값을 대조하여 사용자와 대조하기 위함이다.**