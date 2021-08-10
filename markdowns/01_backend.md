# 1-1. 앱 설계  
django 프로젝트는 여러 app들로 구성되어 있다. 즉, 프로젝트는 하나의 웹사이트이고, 각 앱들은 하나의 웹사이트에 속해있는 기능들이라고 생각하면 된다. e.g) 게시판, 이메일, 결제..  
현재 진행할 주제에는 기능이 **실시간 주가 정보 조회, 주가 정보 예측**이 있다. 고로 두 가지의 앱을 생성할 것이다.  

```
python manage.py startapp stock_inquiry
python manage.py startapp stock_prediction
```  

2020-12-04 : API 서버로 사용할 rest_api 앱도 생성한다.  

```
python manage.py startapp rest_api
```


# 1-2 모델 설계  
- 사용자 정보 모델 (즐겨찾기 종목 포함)  

```python
class Post(models.Model):
    user_name = models.CharField(max_length=200)
    # bookmark_item_list = models.CharField(max_length=200) # 연구중..

    def __str__(self):
        return self.user_name
```  

- 주식시장 모델  

```python
class StockMarket(models.Model):
    stock_market_name = models.CharField(primary_key=True, max_length=200)

    def __str__(self):
        return self.stock_market_name
```

- 주식종목 목록 모델
```python
class StockItemList(models.Model):
    stock_market_name = models.ForeignKey(StockMarket, on_delete=models.CASCADE)
    stock_item_name = models.CharField(primary_key=True, max_length=200)
    stock_item_code = models.CharField(max_length=200)

    def __str__(self):
        return self.stock_item_name
```

- 주식종목 모델  

```python
class StockItem(models.Model):
    stock_item_name = models.ForeignKey(StockItemList, on_delete=models.CASCADE)
    reg_date = models.DateField(default=timezone.now(), null=True)
    high = models.FloatField(default=0.0)
    low = models.FloatField(default=0.0)
    open = models.FloatField(default=0.0)
    close = models.FloatField(default=0.0)
    volume = models.FloatField(default=0.0)
```

# 1-3. REST API 구현 (DRF 적용)  
DRF는 왜 쓸까? (참고 : https://medium.com/@whj2013123218/django-rest-api%EC%9D%98-%ED%95%84%EC%9A%94%EC%84%B1%EA%B3%BC-%EA%B0%84%EB%8B%A8%ED%95%9C-%EC%82%AC%EC%9A%A9-%EB%B0%A9%EB%B2%95-a95c6dd195fd)  
1. 기존의 native django 방식대로 개발을 한다면 프론트 부분은 백엔드로부터 데이터를 받고 django template에 개발을 해야할 것이다. 이럴 경우, 백엔드와 프론트의 완전한 분리가 어렵다. 그래서 DRF를 사용하면 rest api가 사용가능하기 때문에 **django 백엔드와 react 프론트가 분리가 가능**하다.  
2. 재사용성이 좋아진다. view에서 바로 html로 넘기게 되면 view에는 비슷한 로직도 매번 class-based view로 작성해야 하는 비효율적인 상황이 연출된다. 그러나 api를 적용하면 해당 **api를 재사용**할 수 있다. 

```
pip install djangorestframework
```  

DRF를 설치한 후 settings.py에 다음과 같이 적어준다.  

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'stock_inquiry',
    'stock_prediction',
    'rest_framework', # DRF를 앱으로 등록
    'rest_api' # api 서버로 사용할 앱
]

REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
    ]
}
```  

rest_api앱에 urls.py를 생성한 뒤, 루트 urls.py에 다음과 같은 경로를 라우팅한다.  

```python
urlpatterns = [
    ...
    path('rest_api/', include('rest_api.urls')),
]
```  

그 다음 해당 모델을 serialize해야 한다.  
그 이유는 Django ORM의 Queryset은 Context로써 Django template으로 넘겨지며, HTML로 렌더링되어 Response로 보내지게 된다.
하지만 **JSON으로 데이터를 보내야 하는 RESTful API**는 HTML로 렌더링 되는 Django template를 사용할 수 없다. 그래서 Queryset을 Nested한 JSON
으로 매핑하는 과정을 거쳐야 하기 때문이다. (Queryset >> Json : Serialize)  

rest_api앱에 serializers.py를 작성해보자.  

```python
from rest_framework import serializers
from django.contrib.auth.models import User

from .models import StockUser
from .models import StockMarket
from .models import StockItemList
from .models import StockItem



class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'username', 'email')


class StockUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = StockUser
        fields = '__all__'


class StockMarketSerializer(serializers.ModelSerializer):
    class Meta:
        model = StockMarket
        fields = '__all__'


class StockItemListSerializer(serializers.ModelSerializer):
    class Meta:
        model = StockItemList
        fields = '__all__'


class StockItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = StockItem
        fields = '__all__'

```  

model과 serializer를 완성시켰으니 이제 view를 작성할 차례이다.  

(참고 : https://ssungkang.tistory.com/entry/Django-APIView-Mixins-generics-APIView-ViewSet%EC%9D%84-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90)  

DRF View 종류  
- CBV(Class-Based View, APIView 상속)
- FBV(Function-Based View, @api_view 데코레이터 사용)
- Mixin(요청마다 serializer 정의하는 것을 최소화)
- generic APIView
- ViewSet

rest_api/views.py  

```python
from django.http import HttpResponse
from django.shortcuts import render

from rest_framework import viewsets 

from .serializers import StockUserSerializer
from .serializers import StockMarketSerializer
from .serializers import StockItemListSerializer
from .serializers import StockItemSerializer


from .models import StockUser
from .models import StockMarket
from .models import StockItemList
from .models import StockItem


# Create your views here.
class StockUserViewSet(viewsets.ModelViewSet):
    queryset = StockUser.objects.all()
    serializer_class = StockUserSerializer


class StockMarketViewSet(viewsets.ModelViewSet):
    queryset = StockMarket.objects.all()
    serializer_class = StockMarketSerializer


class StockItemListViewSet(viewsets.ModelViewSet):
    queryset = StockItemList.objects.all()
    serializer_class = StockItemListSerializer

    def get_queryset(self):
        stock_market_name = self.request.query_params.get('stock_market_name', None)
        if stock_market_name is not None:
            self.queryset = self.queryset.filter(stock_market_name=stock_market_name)
        return self.queryset


class StockItemViewSet(viewsets.ModelViewSet):
    queryset = StockItem.objects.all()
    serializer_class = StockItemSerializer

    def get_queryset(self):
        stock_item_name = self.request.query_params.get('stock_item_name', None)
        if stock_item_name is not None:
            self.queryset = self.queryset.filter(stock_item_name=stock_item_name)
        return self.queryset

```  

url을 라우팅 시켜보자. viewset을 라우팅할 때에는 CBV, FBV, Mixin, GenericAPIView와는 다르게 router객체로 간편하게 등록할 수 있다.  
rest_api/urls.py  

```python
from django.urls import path, include

from rest_framework.routers import DefaultRouter

from .views import StockUserViewSet
from .views import StockMarketViewSet
from .views import StockItemListViewSet
from .views import StockItemViewSet

router_user = DefaultRouter()
router_user.register('user', StockUserViewSet)

router_market = DefaultRouter()
router_market.register('market', StockMarketViewSet)

router_item_list = DefaultRouter()
router_item_list.register('item_list', StockItemListViewSet)

router_item = DefaultRouter()
router_item.register('item', StockItemViewSet)

urlpatterns = [
    path('auth/', include('rest_framework.urls', namespace='rest_framework')),

    path('', include(router_user.urls)),
    path('', include(router_market.urls)),
    path('', include(router_item_list.urls)),
    path('', include(router_item.urls)),
]
```  

서버를 실행시킨 후, http://localhost:8000/rest_api/posts/에서 자세한 내용을 확인할 수 있다.  
DRF에 관련된 내용은 drf_tutorial을 확인하면 된다.  

위와 같은 방법으로 재사용이 가능한 어떤 REST API를 구현할까? **각 테이블(모델)에 CRUD API**  

# 1-4. URL Patterns
현재 DRF의 ViewSets를 통해 요청 CRUD URL을 구현해놓은 상태이다. 그러나 코드의 추상화 수준이 너무 높아서 문서로 직접 정리를 하려고 한다.  

- 사용자 목록 조회
[GET] rest_api/user/  

- 사용자 추가  
[POST] rest_api/user/  

```
{
    "user_name": "Test"
}  
```  

- 특정 사용자 조회  
[GET] rest_api/user/[id]  

- 특정 사용자 정보 업데이트  
[PUT] rest_api/user/[id]  

```
{
    "id": 10,
    "user_name": "Test U"
}
```  

- 특정 사용자 제거  
[DELETE] rest_api/user/[id]  

사용자 이외의 다른 모델(market, item)도 같은 URL Pattern을 가진다.  