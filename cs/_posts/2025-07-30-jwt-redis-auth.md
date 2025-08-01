# 📌 Summary

<hr>

JWT + Redis 조합의 사용자 Auth 구현 방법을 기록한다. 

<br><br><br><br><br>

# 📌 JWT란?

<hr>

JWT(JSON Web Token)는 네트워크에서 장치간 통신(클라이언트 서버, P2P 등) 시에 사용되는 인증 방식이다. 
JSON 객체로 된 토큰을 사용하는 개방형 표준으로 비밀키와 HMAC 알고리즘으로 서명을 생성한다.

<br><br><br>

## ✨ JWT의 구조(Header, Payload, Signature)

JWT는 JSON(JavaScript Object Notation) 형태로 이루어진 웹 토큰이다 `.`으로 구분된 세 부분으로 구성되며, 각각 헤더(Header), 페이로드(Payload), 서명(Signature) 역할이다.

``` json
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.   ← 헤더 (Header)
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.   ← 페이로드 (Payload)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← 서명 (Signature)
```

<br> 

이는 Base64Url로 인코딩된 문자열 형태이며 아래와 같은 역할이다.

<br>

<b>🍀 헤더(Header)</b>

헤더는 토큰의 메타데이터를 담는다. 
 - alg(Algorithm): 토큰 서명에 사용될 암호화 알고리즘을 지정(HS256, RS256 등)
 - typ(Type): 토큰의 타입을 지정(대부분 JWT)

``` json
 {
    "alg": "HS256",
    "typ": "JWT"
  }
```

<br>

<b>🍀 페이로드(Payload)</b>

페이로드는 토큰에 담고자 하는 실제 정보, 즉 클레임(Claim)을 포함하는 부분이다. 클레임은 사용자 ID, 권한, 생성 시간 등 서버와 클라이언트 간에 공유될 정보들을 Key-Value 형태로 담는다. 페이로드의 정보는 인코딩될 뿐 암호화되지 않는다. 따라서 민감한 정보(비밀번호, 개인 식별 정보 등)는 제외해야 한다.

다음과 같은 정보를 포함할 수 있는데, 개인적으로 `sub`, `exp`만 포함하는 것을 선호한다. 
- iss (Issuer): 토큰 발행자
- exp (Expiration Time): 토큰 만료 시간 (Unix Time)
- sub (Subject): 토큰 제목 (주로 사용자 ID)
- aud (Audience): 토큰 수신자
- iat (Issued At): 토큰 발행 시간

<br>

<b>🍀 서명(Signature)</b>

서명은 무결성 검증을 위한 핵심 요소이다. 토큰이 송수신 과정에서 변조되었는지 확인하는 역할이다. 서명은 다음 세 가지를 결합하여 생성된다. 

1. 인코딩 헤더
2. 인코딩 페이로드
3. 서버만 아는 비밀 키

이 세 가지 값을 JWT 헤더에 지정된 암호화 알고리즘을 사용하여 해싱한다. 
주로 대칭 키 암호화 방식인 HMAC(Hash-based Message Authentication Code) 계열의 알고리즘(HS256, HS512 등)을 사용한다. 


HS256은 SHA256 해시 함수를 기반으로 HMAC을 적용한 알고리즘이며 특정 비밀 키를 사용하여 입력 데이터의 해시 값을 생성한다.  이렇게 생성된 메시지 인증 코드(MAC)를 통해 데이터의 무결성과 원본을 인증할 수 있다.

``` json
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secretKey
)
```

<br><br><br>

## ✨ 세션과 JWT 비교

<b>🍀 세션의 장점</b>

세션은 사용자의 인증 정보를 서버측에만 저장한다. 
클라이언트 측에서는 사용자의 세션 ID만을 갖고 있으며 실제 데이터는 서버에 위치하기 때문에 상대적으로 안전하다는 장점이 있다.
또한 세션 ID 탈취 등 보안 이슈가 발생했을 때 서버 측에서 해당 세션을 직접 무효화시키는 것이 가능하다. 

<br>

<b>🍀 세션의 단점</b>

서버가 확장되어 있을 경우 서버끼리의 세션 정보 공유를 위한 추가적인 개발 및 설정이 필요하다. 
매 요청마다 세션 저장소를 조회해야 한다. 사용자 수가 많을수록 서버의 부하가 심해진다. 

<br>

<b>🍀 JWT의 장점</b>

`JWT`는 토큰을 브라우저와 같은 클라이언트 측에 저장한다. 
`Stateless`하게 동작할 수 있어 수평 확장에 용이하며 
토큰 자체에 (탈취당해도 되는)사용자 정보를 포함할 수 있다. 

<br>

<b>🍀 JWT의 단점</b>

서명으로 무결성은 보장할 수 있지만 페이로드는 디코딩하면 누구나 확인 가능하다. 
한번 발급된 이후에는 토큰이 만료되기 전까지 서버에서 무효화하기 어렵다. 
(`JWT`만 사용하여 인증을 구현하면 사실상 무효화 방법이 없다고 생각한다.)

<br>

<b>🍀 비교</b>

요약하자면, <br>
세션은 무효화가 가능하지만 `JWT`는 무효화가 어렵다. <br>
세션은 정보를 서버측에서 관리하기 때문에 안전하지만, `JWT`는 토큰에 정보가 일부 포함된다. <br>
세션은 서버에 부하가 발생할 수 있지만, `JWT`는 부하 이슈가 없어서 확장에 용이하다. <br>

<br><br><br>

## ✨ 액세스 토큰과 리프레쉬 토큰의 발급 및 인증 절차

`JWT`는 액세스 토큰(Access Token)과 리프레쉬 토큰(Refresh Token) 두개의 토큰 유형이 존재한다. 왜 토큰이 두개로 구분되어 있을까? 🤔 

<br>

<b>🍀 액세스 토큰</b>

액세스 토큰은 말 그대로 리소스에 접근하기 위한 인증 용도이다. 
짧은 유효기간을 부여하여 탈취 시 피해를 최소화한다. 
`HTTP` 요청 시 `헤더`에 토큰을 포함하여 권한이 있는 사용자임을 증명한다. 

<br>

<b>🍀 리프레쉬 토큰</b>

리프레쉬 토큰은 액세스 토큰이 만료되었을 때, 
재로그인을 요구하지 않고 새로운 액세스 토큰을 발급하기 위해 사용된다. 
액세스 토큰보다 긴 유효기간으로 로그인 상태를 유지하기 위한 용도이다. 
리프레쉬 토큰은 액세스 토큰과 달리 서버에서도 저장 및 관리해야 한다. 
유효하지 않거나 변조된 토큰을 블랙 리스트 처리하여 무효화하기 위함이다. 
또한 동시 로그인 또한 제한할 수 있다. 

<br>

<b>🍀 토큰 발급 및 인증 절차</b>

아래의 절차 이미지를 참고하여 사용자 플로우대로 설명하겠다. 
 
<img src="https://developers.worksmobile.com/image/kr/api/authorization/auth_service_account.svg">

<br>

`1. Get Access Token` 

1. 사용자는 브라우저(클라이언트)의 로그인 페이지에서 아이디와 비밀번호를 입력한다. 
2. 로그인 버튼 클릭 시 서버에 `POST` 메서드로 로그인 요청이 전송된다. 
3. 서버는 유효한 사용자인지 판단한 이후 토큰(액세스 토큰, 리프레쉬 토큰)을 발급한다. <br>(응답 바디에 두가지 토큰을 모두 포함하며  리프레쉬 토큰은 서버 DB에 저장하여 관리한다.)
4. 클라이언트는 발급받은 토큰을 `쿠키`, `스토리지` 등 저장소에 저장한다. 

<br>

`2. API Request`

1. 사용자 인증이 필요한 기능에는 `API` 요청 시 헤더에 토큰값을 담아서 요청한다. 
2. 서버는 유효한 토큰인지 판단한 이후 클라이언트에게 응답한다.<br>
(유효하지 않다면 `HTTP 401 Unauthorized`과 같은 응답을 받은 클라이언트가 `Refresh Access Token` 단계를 진행할 것이다.)

<br>

`3. Refresh Access Token`

1. 서버로부터 액세스 토큰이 유효하지 않다는 응답을 받은 클라이언트는 리프레쉬 토큰을 전달한다. 
2. 서버는 리프레쉬 토큰의 유효성을 검증한 이후 클라이언트에게 새로운 액세스 토큰을 발급해준다. <br>(리프레쉬 토큰이 유효하지 않다면 재로그인을 유도하는게 일반적이다.)

<br><br><br><br><br>

# 📌 Redis란?

<hr>

## ✨ Redis의 특징

`Redis`는 인메모리 데이터 구조 저장소이며 `NoSQL` 데이터베이스이다. <br>
(Redis는 다른 포스팅에서 자세하게 한번 다뤄야겠다. 👀)

- 인메모리 저장방식: 디스크에 데이터를 저장하는 `MySQL`, `MSSQL` 등의 DB와 달리 메모리에 데이터를 저장하기 때문에 Read/Write 속도가 매우 빠르다. 
- 다양한 자료구조: 문자열, 해시, 리스트, 셋, 정렬된 셋(Sorted Set) 등 다양한 데이터 구조를 지원한다.
- 영속성 지원: 기본적으로 메모리에 데이터를 적재하지만, 디스크에도 데이터를 저장할 수 있는 `AOF(Append-Only File)`와 `RDB(Redis Database)` 스냅샷 옵션을 제공하여 영속성을 가질 수 있다. 
- 단일 스레드 모델: 단일 스레드로 원자적 연산을 보장한다. (Redis 6.0부터 멀티 스레드 I/O 지원)
- TTL(Time-To-Live) 지원: 키별 만료 시간을 설정할 수 있어 캐시로 활용할 수 있다.

<br>

인메모리라는 장점을 이용하여 다양하게 활용할 수 있다. (`Caching`, `Session Storage`, `Message Queue` 등)

<br><br><br>

## ✨ JWT + Redis의 장점

1. 리프레시 토큰 저장 <br>
앞서 말했듯 토큰 발급 이후 리프레쉬 토큰은 서버에서 저장하고 관리해야 한다. <br>
Redis에 리프레쉬 토큰을 저장하여 빠른 I/O 처리로 토큰 검증 과정에서 성능을 향상시킬 수 있다. <br>

2. 토큰 탈취 시 유효하지 않은 요청이 수신될 것이다. 이때 Redis에 저장된 리프레쉬 토큰을 무효화 처리(블랙 리스트)하여 보안 사고를 방지할 수 있다.

이외에도 스케일 아웃(Scale-Out)이 가능하고 데이터베이스에 부하를 줄이는 등 성능과 관련한 장점들이 존재한다. 

<br><br><br><br><br>

# 📌 구현

<hr>

`Python`과 `Fast API`로 개발하였다. 

<br><br><br>

## ✨ FastAPI 실행

Fast API 서버를 오픈하고 Redis 설정을 호출한다. (DB Init 과정은 생략)

``` Python
# main.py

from fastapi import FastAPI
from app.apis import auth, user, post
from app.core.redis_config import init_redis

app = FastAPI()

app.include_router(user.router, tags=["user"])
app.include_router(auth.router, tags=["auth"])
app.include_router(post.router, tags=["post"])

init_redis(app)
...
```

<br><br><br>

## ✨ Redis 설정

<br> Redis 설정(`decode_responses=True`는 데이터를 문자열로 가져오는 설정이다.)

``` Python
# app/core/redis_config.py

from fastapi import FastAPI
import redis
import redis.exceptions

REDIS_HOST = "localhost"
REDIS_PORT = 6379
REDIS_DB = 0
REDIS_PASSWORD = None

redis_client = redis.Redis(
    host = REDIS_HOST, 
    port=REDIS_PORT, 
    db=REDIS_DB,
    password=REDIS_PASSWORD,
    decode_responses=True
)

def init_redis(app: FastAPI):
    @app.on_event("startup")
    async def startup_redis_client():
        try:
            redis_client.ping()
        except redis.exceptions.ConnectionError:
            print("Failed to connect to Redis")

    @app.on_event("shutdown")
    async def shutdown_redis_client():
        redis_client.close()
```

<br><br><br>

## ✨ Auth API 구현

``` Python
# app/apis/auth.py

router = APIRouter()
bearer_scheme = HTTPBearer()

@router.post("/login")
def login(login_data: LoginRequest, auth_service: AuthService = Depends(get_auth_service)):
    user = auth_service.authenticate_user(login_data)

    if not user:
        raise HTTPException(
            status_code=401,
            detail="인증 실패", 
            headers={"WWW-Authenticate": "Bearer"}
        )
    
    token_data = auth_service.create_user_token(user)
    return token_data

@router.post("/logout")
def logout(credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme)):
    token = credentials.credentials
    token_expiry = get_token_expiry(token)
    #print(token_expiry)
    #print("------------")
    TokenService.blacklist_token(token, token_expiry)

    return {"message": "로그아웃"}

@router.post("/refresh", response_model=TokenResponse)
def refresh_token(refresh_token: RefreshRequest, auth_service: AuthService = Depends(get_auth_service)):
    token = auth_service.refresh_access_token(refresh_token.refresh_token)

    if not token:
        raise HTTPException(
            status_code=401,
            detail="사용자 접근이 유효하지 않습니다.",
            headers={"WWW-Authenticate": "Bearer"}
        )
    
    token["refresh_token"] = refresh_token.refresh_token
    return token

@router.post("/logout-all")
def logout_all_sessions(credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme)):
    token = credentials.credentials
    token_expiry = get_token_expiry(token)
    TokenService.blacklist_token(token, token_expiry)

    user_id = verify_token(token).get("user_id")
    TokenService.remove_refresh_token(user_id)

    return {"message": "모든 기기로부터 로그아웃되었습니다."}
```

위에서 조금 중요한 부분은 `Depends()` 구문이다. <br>
Fast API는 요청이 들어오면 해당 경로(함수)를 확인하고 `Depends()`가 있으면 먼저 실행하여 의존성을 얻는다. <br>
`Depends()` 안에 있는 함수의 결과값을 원래의 매개 변수로 전달한다. 

<br>

로그인 API를 예시로 로직을 설명하자면, 
``` Python
def login(login_data: LoginRequest, auth_service: AuthService = Depends(get_auth_service))
```
1. `POST /login`과 같은 HTTP 요청을 받게 되면 `login_data: LoginRequest` 구문을 통해 Request body에 데이터가 정상적으로 들어왔는지 `LoginRequest` `Pydantic` 모델의 유효성을 판단한다. 
2. 유효하다면 `auth_service: AuthService = Depends(get_auth_service)`을 실행한다. 
3. `Depends()`는 의존성을 갖기 위한 구문인데 괄호 안에 있는 `get_auth_service`를 먼저 실행한다. 
4. `get_auth_service`의 결과값으로 AuthService 클래스의 인스턴스를 생성하여 반환한다.

<br>

요약: <br>
`LoginRequest` 클래스를 `Pydantic` 모델로 유효성을 검증하고, 유효하다면 `Depends`를 통해 `AuthService` 인스턴스를 생성받는다.
