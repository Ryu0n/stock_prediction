# 4-1 뷰 설계  
## 4-1-1 실시간 주가 정보 조회앱의 뷰 설계  

- 주식시장 선택 뷰 (무한스크롤형)  
각 주식시장별 그래프를 띄우며, 해당 주식시장(코스피200, 코스닥150)의 종목별 주가 정보 조회 뷰로 접근할 수 있게 해준다.  
무한스크롤형을 선택한 이유는 주식시장이 추가되는 경우를 고려한 것이다.  

```javascript
import React, { Component } from 'react'
import CardStockMarket from '../card/cardStockMarket'

class ListStockMarket extends Component {
    render() {
        var markets = [];
        var marketDatas = this.props.data;
        var i = 0;

        for (i = 0; i < marketDatas.length; i++){
            marketData = marketDatas[i]
            markets.push(
            <li key = {marketData.id}>
                <CardStockMarket name={marketData.name}></CardStockMarket>
            </li>
            );
        }

        return (
        <nav>
            <ul>
            {markets}
            </ul>
        </nav>
        );
    }
  }

  export default ListStockMarket;

```

```javascript
// 리스트의 각 항목의 역할을 하는 카드 컴포넌트
import React, { Component } from 'react'

import './cardStockMarket.css'

import {Card, CardHeader, CardContent, CardActions, CardMedia} from '@material-ui/core'
import Typography from '@material-ui/core/Typography';
import Button from '@material-ui/core/Button'

class CardStockMarket extends Component {
    render() {
        var marketName = this.props.name;
        
        return (
            <Card className="root">
                <CardHeader>  
                </CardHeader>
                <CardContent>
                    <Typography className="content">I'm {marketName}. </Typography>
                </CardContent>
                <CardActions>
                    <Button size="small">GO TO</Button>
                </CardActions>
            </Card>
        );
    }
  }

  export default CardStockMarket;

```  
  

**2020.12.29**  
각 주식시장을 표현할 카드 설계중   
<img width="1440" alt="스크린샷 2020-12-29 오후 10 46 30" src="https://user-images.githubusercontent.com/32003817/103288259-d30da980-4a27-11eb-8a1c-fdfab8a6fe86.png">

**2021.02.24**  
![image](https://user-images.githubusercontent.com/32003817/108954001-750be400-76af-11eb-94f8-31755fea7d8a.png)
참고 : https://www.zerocho.com/category/React/post/579b5ec26958781500ed9955  
mongodb로부터 가져온 주식시장 정보를 바탕으로 CardStockMarket 컴포넌트를 출력한 내용이다. 우선 React 컴포넌트의 라이프사이클에 대한 간단한 설명을 하자면, 컴포넌트가 최초로 실행되면 mount된다고 표현한다. 이 대, context, defaultProps와 state를 저장한다. 그리고 **componentWillMount** 메소드가 호출된다. 이때는 컴포넌트가 DOM에 부착되기 전이므로 state나 props를 바꿔선 안된다. **render** 메소드를 통해 DOM에 부착된 후, **componentDidMount** 메소드가 호출된다. componentDidMount 메소드는 컴포넌트가 최초로 실행되고 DOM에 부착된 후에 실행되므로 각 컴포넌트가 생성된 후 최초 한번만 호출된다. 이때는 DOM에 부착되어 있는 상태이기 때문에 AJAX요청이나 setTimeout, setInterval과 같은 행동을 한다.  

그리고 axios 모듈을 통해 react로부터 django 서버를 거쳐 mongodb에 있는 데이터를 요청해왔다. python의 requests 모듈과 같다고 생각하면 된다. **아직 원인은 파악하지 못했지만 render에서 axios를 통해 요청할 경우 2번씩 요청되는 현상이 발생했다.** componentDidMount 라이프사이클에서 요청하자 이와같은 문제는 해결됬다.  
```javascript
// App.js 

...

  componentDidMount(){
    console.log('im didmount!');
    axios({
      method : "get",
      url : "http://127.0.0.1:8000/rest_api/market/"
    })
    .then(response => {
      console.log(response);
      let _markets = [];
      response.data.forEach(element => {
        let _id = element["id"];
        let _name = element["stock_market_name"];
        let market = <CardStockMarket name={_name}/>
        _markets = _markets.concat(market);
      });
      console.log('hello', _markets);
      this.setState({markets: _markets});
    })
    .catch(error => {
      console.log("error", error);
    })
  }
```
setState를 함으로써 render를 한번더 유발시킨다. 이때 우리가 요청한 데이터를 토대로 카드를 화면에 출력하게 된다.  

**2021.03.07**  
```javascript
// App.js

class App extends Component{

  state = {
    view_mode: 'market',
  }

  constructor(props){
    super(props);
  }

  // shouldComponentUpdate(){
  // }

  render(){
    let template = null;
    if (this.state.view_mode == 'market') {
      template = <ListStockMarket/>;
    }

    return (
      <div className="App">
        {template}
      </div>
    );
  }
}
```

```javascript
// listStockMarket.js

class ListStockMarket extends Component {
    state = {
        markets: []
      }
    
    constructor(props){
    super(props);
    }

    render(){
        return (
          <div id="grid">
            <ul className="stockMarkets">
              {this.state.markets}
            </ul>
          </div>
        );
      }
    
    componentDidMount(){
    axios({
        method : "get",
        url : "http://127.0.0.1:8000/rest_api/market/"
    })
    .then(response => {
        console.log(response);
        let _markets = [];
        response.data.forEach(element => {
        let _id = element["id"];
        let _name = element["stock_market_name"];
        let market = <CardStockMarket name={_name}/>
        _markets = _markets.concat(market);
        });
        this.setState({markets: _markets});
    })
    .catch(error => {
        console.log("error", error);
    })
    }
}
```
App.js 의 App 클래스에 view_mode 속성을 지닌 state를 세팅함으로써 상황에따라 동적으로 보여질 컴포넌트를 렌더링할 것이다. 기존의 로직은 listStockList.js로 옮기면서 구조를 리팩토링했다.
  
**2021.03.14**  
이번 시간에는 주식시장에 해당하는 주식종목만 가져오는 것이 목표이다. 그러나 http://127.0.0.1:8000/rest_api/item/ url로 GET 요청을 보내면 조건에 관계없이 종목을 전부 가져오는 불상사가 일어난다. 그래서 우리는 특정 조건을 설정해줄 필요가 있다.  

```python
class StockItemViewSet(viewsets.ModelViewSet):
    queryset = StockItem.objects.all()
    serializer_class = StockItemSerializer

    def get_queryset(self):
        reg_date = self.request.query_params.get('reg_date', None)  # 쿼리스트링
        if reg_date is not None:
            self.queryset = self.queryset.filter(reg_date=reg_date)
        return self.queryset
```
위 코드는 viewset으로 구현된 요청 url에 쿼리스트링을 받아 모델로부터 해당하는 데이터만 queryset으로 반환한다. 테스트로 등록일자만 쿼리해보았다.  
![image](https://user-images.githubusercontent.com/32003817/111063782-f529aa80-84f3-11eb-9312-37b39e857e12.png)

- 종목별 주가 정보 조회 뷰 (List)  
현재 시점 해당 주식시장의 여러 종목의 주가를 조회 가능.  
해당 종목에 하이퍼링크를 달아 상세 정보 조회 뷰로 넘어갈 수 있게 한다.  

- 해당 종목 상세 정보 조회 뷰 (Detail)  
해당 종목의 주가를 기간별로 조회 가능 (그래프 적용)  
e.g) 1년전, 6개월전, 3개월전, 1개월전  

~~- 기능 패널  
모든 뷰의 우측에 패널로써 존재. 해당 패널에는 기능(Util)들이 속해있다.  
깔끔한 디자인을 위해 무한스크롤형 리스트에 기능을 추가한다. (기능의 확장 고려)~~

~~- 즐겨찾기 관리    
기능 패널에 속해있는 기능.  
사용자가 등록해둔 종목의 상세 정보 조회 뷰로 이동할 수 있게 한다.~~

~~- 종목별 주가 비교  
기능 패널에 속해있는 기능.  
종목별 주가 비교 뷰로 이동할 수 있게 한다.~~

~~- 종목별 주가 비교 뷰  
사용자가 원하는 종목을 추가하면 그래프를 겹쳐서 그림~~

## 4-1-2 주가 정보 예측앱의 뷰 설계  

- 해당 종목 주가 예측 뷰  
해당 종목의 예측주가를 그래프로 시각화  
종목별로 뷰가 동일할 것으로 예상되므로 **템플릿** 적용