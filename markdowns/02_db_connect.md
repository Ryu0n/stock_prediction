# 2-1. MongoDB 연동 절차
1. MongoDB 가입 -> 클러스터 생성 이후 하단의 그림과 같이 인증방식, DB 접근권한 등을 설정하여 Database user를 생성

![db1](https://user-images.githubusercontent.com/36228833/103440836-3dbf1f00-4c8c-11eb-8713-8c96c20b1b1c.PNG)

![db2](https://user-images.githubusercontent.com/36228833/103440843-4c0d3b00-4c8c-11eb-8fba-5fa45642e147.PNG)

2. 왼쪽 메뉴에 Network Access에 들어간 뒤 DB에 접근을 허용할 IP를 등록

![db3](https://user-images.githubusercontent.com/36228833/103440947-6562b700-4c8d-11eb-904e-4935f37f653e.PNG)

![dbdbdb](https://user-images.githubusercontent.com/36228833/103440950-6eec1f00-4c8d-11eb-9184-605af1f29c62.PNG)

**(스킵) Django 모델을 migration하는 과정에서 데이터베이스가 자동으로 생성되므로 스킵

![db4](https://user-images.githubusercontent.com/36228833/103440978-bb375f00-4c8d-11eb-9242-d394583a181c.PNG)

3. 첫 페이지로 돌아와서 현재 Cluster의 Connection string을 복사

![db5](https://user-images.githubusercontent.com/36228833/103440984-d1ddb600-4c8d-11eb-8ef9-97737853dfdc.PNG)

![db6](https://user-images.githubusercontent.com/36228833/103440994-ef128480-4c8d-11eb-9ef4-82e1bcbc3851.PNG)

4. connection string으로 mongoDB 연동을 위한 모듈들을 설치

```
$ pip install dnspython
$ pip install djongo
```

5. 장고 프로젝트 내 settings.py 에 mongoDB 연동을 위한 코드 작성

```python
DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        "CLIENT": {
           "name": "testDB",   # Database 이름입니다.
           "host": "mongodb+srv://admin:<password>@cluster0.lvfxw.mongodb.net/<dbname>?retryWrites=true&w=majority",   # 방금 복사한 Connection string
           "username": "admin",   # Database username
           "password": "admin",   # Database user's password
           "authMechanism": "SCRAM-SHA-1",   # 인증방식
        },
    }
}
```

6. migrate 명령어를 통해 모델의 변동사항을 데이터베이스에 적용

![db9](https://user-images.githubusercontent.com/36228833/103441103-fab27b00-4c8e-11eb-98d6-87eb8c1502ff.PNG)

7. 첫 페이지의 collections를 눌러 DB가 잘 연동되었는지 확인

![db8](https://user-images.githubusercontent.com/36228833/103441074-c212a180-4c8e-11eb-81b5-787bf727f28e.PNG)

# 2-2. settings.py 기밀정보 관리
settings.py 에는 secret_key, DB 정보 등 외부에 노출되면 안되는 기밀정보들을 담고있다.

SECRET_KEY의 용도 -- [출처](https://docs.djangoproject.com/en/3.0/ref/settings/#secret-key)
- [암호화된 데이터 서명](https://docs.djangoproject.com/en/1.11/topics/signing/)을 포함하고 있다.
- 사용자 세션, 비밀번호 재설정 요청, 메시지 등을위한 고유 토큰을 포함하고 있다.

Django 앱에는 암호화 서명이 필요한 많은 것들이 있으며 'SECRET_KEY' 설정이 그 열쇠라고 볼 수 있다. 해당 기밀 정보들을 다음과 같이 json 파일로 관리하여 따로 호출하게끔 변경한다.

* secrets.json
```json
{
    "SECRET_KEY": "your SECRET_KEY",
    "DB_HOST" : "your DB Host",
    "DB_NAME" : "your DB name",
    "DB_USERNAME" : "your DB Username",
    "DB_PASSWORD" : "your DB Password"
}
```

* settings.py
```python
secret_file = os.path.join(BASE_DIR, 'secrets.json')

with open(secret_file) as f:
    secrets = json.loads(f.read())

def get_secret(setting, secrets=secrets):
    """
    secrets.json을 통해 값을 가져온다.
    """
    try:
        return secrets[setting]
    except KeyError:
        error_msg = "secrets.json 파일에 {} 값이 존재하지 않습니다.".format(setting)
        raise ImproperlyConfigured(error_msg)

SECRET_KEY = get_secret("SECRET_KEY")
```