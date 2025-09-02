---
layout: post
title: 성능 테스트로 견고한 서버 애플리케이션 개발하기(With Artillery)
description: >
  Artillery로 서버 애플리케이션의 성능 테스트 방법에 대한 포스팅입니다.
sitemap: true
hide_last_modified: true
---

* toc
{:toc .large-only}

<br><br><br><br><br>

# 📌 Overview

<hr>

서비스 개발에서 빠질 수 없는 것이 테스트이다. 
완성된 결과물에서 예외가 발생하지는 않는지, 
동시 몇 명의 사용자까지 수용할 수 있는지를 검증하기 위해 테스트를 진행해야 한다. 

테스트를 위해서 어떠한 상황에서 서버에 부하가 발생하고, 
부하가 발생하는 근본적인 원인이 무엇인지, 
이를 어떻게 개선할 수 있는지 알아야 한다. 

성능 테스트와 관련하여 학습한 전반적인 내용과 실제 Artilery를 사용한 테스트 내용을 기록한다. 




<br><br><br><br><br>

# 📌 성능 테스트란

<hr>

## ✨ 성능 테스트의 목적

서버에 부하가 발생하는 상황을 가정해 보자. 
우리는 신규 서비스 오픈을 앞두고 있다. 
예상 사용자는 100명 내외로 매우 적은 트래픽이 예상되었다. 
하지만 막상 오픈하니 기대 이상으로 많은 사용자가 방문하여 트래픽 또한 늘어나게 되었다. 
서버는 요청을 정상적으로 처리하지 못해 `Latency`가 증가하였고 일부 요청에서는 `Time Out`이 발생하게 된다. 

<br>

이때 부하라는 것은 왜 발생하는 것일까? 
서버는 요청을 처리하기 위해 한정된 자원을 사용한다. 
자원이란 서버 프로세스에 할당된 CPU, 메모리, 디스크 등으로 물리적인 항목이며 무한하지 않다. 
따라서 한 번에 많은 처리하기 위해 특정 자원의 사용 비중이 높아지게 되면 병목 구간, 즉 부하가 발생하게 된다. 

<br>

단순히 생각했을 때 요청이 많으면 서버를 증설하면 된다. 
서버를 증설하고 한 뎁스 위에 로드 밸런서를 구축하여 트래픽을 분산시키면 해결될 문제이다. 
하지만 서버를 얼마나 증설해야 하는지를 알 수 없다. 
(인프라 투자 비용도 한정적이기 때문에 무한정 증설할 수 없다.) 
<b>이러한 상황에서 성능 테스트가 필요한 것이다. </b>
사용자가 많은 상황을 가정해 주는 것이 성능 테스트 툴의 역할이며, 
리포팅 결과를 통해 사용자 요청 수에 따라 얼마만큼의 서버 증설이 필요한가를 도출할 수 있다.

<br><br><br>

## ✨ 다양한 서버의 부하 발생 케이스

서버에 부하가 발생할 수 있는 대표적인 케이스를 살펴보자.

### 🌊 한정된 대역폭으로 인한 Latency 증가

대역폭이 100 MB/s라면 1초에 100MB의 데이터를 전송할 수 있다. 
만약 한 대의 클라이언트가 서버에 접속하여 100MB/s의 데이터를 송수신하려는 찰나, 
다른 클라이언트가 추가되었고 서버에 50MB/s의 데이터를 전송했다. 
서버는 대역폭이 100MB/s이지만 전송해야 할 데이터는 150MB/s가 되어 지연이 발생하게 된다. 

우리가 웹 브라우저에서 파일을 내려받을 때 종종 속도가 느려지는 경우가 있다. 
이는 주로 네트워크 대역폭의 대부분이 사용 중인 상태일 때 발생한다. 

<br>

### 🌊 데이터베이스에서 발생하는 Latency 증가

- 데이터양에 따른 조회 시간: 조회해야 할 데이터의 양이 많아질수록 조회 시간이 길어져 `Latency` 증가
- 대량 데이터 응답: 한 번에 많은 양의 데이터를 응답해야 할 경우 `Latency` 증가
- 트랜잭션 락: 트랜잭션 처리 과정에서 데이터에 락(Lock)이 걸리는 경우, 락이 유지되는 시간만큼 `Latency` 증가
- 데드락 발생: 데드락(Deadlock)이 발생하면 데이터베이스의 응답이 완전히 중단되어 `Latency` 증가(최악의 경우🤬)

<br>

### 🌊 데드락과 스레드 풀 고갈로 인한 Latency 증가

데드락 자체는 서버의 물리적인 자원을 직접적으로 많이 소모하지는 않지만, 
스레드 풀이나 커넥션 풀처럼 제한적으로 생성되는 중요한 자원들을 빠르게 고갈시켜 시스템 전체의 응답률을 떨어뜨린다. 

`스레드 풀(Thread Pool)`은 클라이언트 요청을 효율적으로 처리하기 위해 미리 생성된 스레드들을 관리한다. 
요청이 들어오면 풀에서 스레드를 할당하여 처리하고, 작업이 완료되면 스레드를 풀에 반납하여 재사용한다. 
하지만 모든 스레드가 사용 중인 상태일 때는 새로운 요청들은 큐(Queue)에 저장되어 여유 스레드가 생길 때까지 대기한다. 
만약 큐마저 가득 차게 되면, 더 이상 요청을 받을 수 없어 해당 요청은 버려지게 된다. 

따라서 단순히 스레드 풀의 크기를 무작정 늘리는 것은 바람직하지 않다. 
올바른 해결 방법은 로드 밸런싱을 통한 트래픽 분산이라고 생각한다. 


<br><br><br>

## ✨ 서버의 적정 성능과 올바른 테스트 방법

그렇다면 서버 애플리케이션은 어느 정도의 성능이 적정치라고 볼 수 있을까?
어떤 서비스냐에 따라 다르다고 생각한다. 
✈ 항공권 조회 사이트를 생각해 보자. 
검색이 완료되기까지 수초에서 수십 초가 걸리지만 사용자는 이를 수용한다. 

<br>

반면 구글이나 네이버에 무엇을 검색하는 경우 3000m/s 이내로 검색 결과가 표출된다. 
만약 검색 시간이 항공권 조회처럼 수십 초 걸린다면 사용자는 이를 수용하지 못할 것이다. 

<br>

이는 <b>서비스 유형에 따라 사용자가 기대하는 `Latency`가 다르기 때문이다. </b>
궁극적으로 엔지니어는 사용자의 기대 `Latency`를 맞추는 것을 목표로 해야 한다. 
`Time Out`이 발생하지 않아야 하며, `Latency`와 `Throughput`을 동시 목적으로 한 서버를 개발해야 한다. 
(`Throughput`이 늘어나면 서버는 한정된 물리적 자원을 사용해야 하기 때문에 `Latency`도 자연스레 늘어나는 경우가 일반적이다.)

<br>

적정 성능인지 판단을 위해 적합한 테스트 방법은 아래 순서라고 생각한다. 
1. 요청 한 건에 대한 `Latency` 측정한다. 
2. `Throughput`을 높이면서 `Latency`가 증가하는 지점을 탐색한다. 
3. 어떤 부분이 병목 구간인지 가설을 세운다. 
4. 서버 자원 모니터링, 로그 분석을 통해 정확한 병목 지점을 탐색한다. 
5. 로직을 개선하거나 트래픽을 분산시킨다.

<br><br><br>

## ✨ 성능 테스트와 관련한 CS 지식

### 🌊 자원의 상호작용과 역할

서버 애플리케이션에 필요한 CPU, 메모리, 디스크 등 서버 자원은 `OS`에서 관리하고 할당한다. 
따라서 애플리케이션 성능 관점에서 중요한 역할을 하며 각 자원들은 아래와 같이 상호작용한다. 
- 디스크에 있는 바이너리 파일(.exe)을 프로그램이라고 한다. 
- 프로그램을 실행하면 메모리 위에서 실행되는데 이걸 프로세스라고 부른다. 
- 프로세스는 메모리와 코드로 구성되어 있으며 작성된 코드에 따라 메모리를 수동적으로 실행한다.
- CPU는 프로세스를 구동하는 주체이며 정의된 코드를 보고 메모리를 읽기도, 쓰기도 한다.
- 프로세스가 많으면 램 위에 모두 올릴 수 없다. 따라서 물리적인 메모리를 디스크에 저장했다가 불러와서 사용하는 `페이징` 기법을 사용한다. 

<br>

자원의 주요 역할은 아래와 같다
- CPU는 연산(계산, 이미지/영상 인코딩)이나 암/복호화에 주로 사용된다. 
- 메모리는 일반적으로 CPU 사용에 비례하여 사용된다. 인스턴스 대량 생성, 캐싱, 컬렉션 객체 사용 등의 경우에는 램의 사용률이 더 높다. 
- 디스크는 파일 입출력이나, 데이터베이스 작업 시에 사용된다. 

<br><br>

### 🌊 빛의 속도에 따른 네트워크의 물리적 한계

서버 간의 물리적 거리가 멀어질수록 네트워크 통신으로 인한 `Latency`는 증가한다. 
데이터 전송 속도가 빛의 속도라는 물리적 한계에 의해 제한되기 때문이다. 

<br>

빛의 속도는 299,792,458 m/s로 1초에 지구 7.5바퀴를 돌 수 있다. 
1바퀴를 도는 데에는 133.3 m/s가 걸린다.
지구 반대편에 위치한 서버에 요청을 전송하면 응답이 도착하는 데까지 133.3 m/s 이상 소요될 수밖에 없다는 의미이다. 

<br><br><br><br><br>

# 📌 Artillery로 성능 테스트하기

<hr>

앞서 학습한 이론을 토대로 실제 성능 테스트를 진행해보자.
[Artillery](https://www.artillery.io/)를 사용할 것이며 관련된 테스트 환경 및 사용 버전은 아래와 같다. 

> <b>⚙️ 테스트 환경</b> <br>
OS: Windows 10 x64<br>
IDE: Visual Studio Code 1.103.2 <br>
Node.js: v18.13.0 <br>
npm: 8.19.3 <br>
Artillery: 1.7.9

<br><br><br>

## ✨ Artillery란?

[Artillery](https://www.artillery.io/)는 간단하면서 효과적인 성능 테스트 목적의 오픈 소스 라이브러리이다. 
`REST API`를 비롯한 `GraphQL`, `WebSocket` 등 다양한 프로토콜을 지원하며 
`Azure`, `AWS`와 같은 클라우드 기반 서버리스 환경에 테스트를 적용할 수도 있어 확장성이 높다. 

테스트 결과로 생성되는 로우 데이터를 리포트 형식으로 출력해서 시각화할 수 있다. 

<br>

단점은 복잡한 시나리오 작성의 한계이다. 
장바구니에 상품을 추가하고 결제하는 것과 같이 연속된 시나리오를 작성하는 것이 현재로서는 코드를 수정하는 것 외에는 불가하다.
또한 실시간으로 테스트 상황을 파악하기 어렵다. 
진행 상황을 10초마다 텍스트 형태로 보여주기는 하지만 건 바이 건으로 내용을 확인하기 어렵다. 

따라서 복잡한 시나리오나 실시간 테스트 결과를 확인하려면 Grafana, JMeter 등의 툴을 연동해야 한다. 

<br>

그럼에도 수요가 있는 건 간편한 테스트 방법과 확장성 때문일 것이다.



<br><br><br>

## ✨ Artillery 사용 방법

### 🦕 Artillery 설치

라이브러리를 설치해 주고 

```
npm install -g artillery
```

<br>

정상적으로 설치가 되었는지 확인 

```
artillery version
```

```
------------ Version Info ------------
Artillery: 1.7.9
Artillery Pro: not installed (https://artillery.io/pro)
Node.js: v18.13.0
OS: win32/x64
--------------------------------------
```

<br>

이스터에그 커맨드로 귀여운 공룡을 확인할 수 있다.

```
artillery dino
```

```
 ------------
< Artillery! >
 ------------
          \
           \
            __
           / _)
    .-^^^-/ /
 __/       /
<__.|_|-|_|
```

<br><br>

### 🌊 테스트 서버 작성

간단하게 서버를 구성했다. `login`과 `login-delay` 두 개의 API를 구성했고 `login-delay`는 응답 전 3초의 지연이 발생한다.

``` javascript
// index.js
const express = require('express');
const app = express();
const port = 8080;

app.use(express.json());
app.use(express.urlencoded({ extended: true }));



app.post('/login', (req, res) => {
    const { id, pw } = req.body; 
    if (!id || !pw) return res.status(400).json({ message: 'ID 또는 PW 미입력' });
    console.log(`ID: ${id}, PW: ${pw}`);
    res.status(200).json({ message: '로그인 성공'});
});



app.post('/login-delay', (req, res) => {
  const { id, pw } = req.body;
    if (!id || !pw) return res.status(400).json({ message: 'ID 또는 PW 미입력' });
    setTimeout(() => {
        res.status(200).json({ message: '지연 로그인 성공'});
        console.log(`/login-delay API 응답 완료 - ID: ${id}, PW: ${pw}`);
    }, 3000);
});




app.listen(port, () => {
  console.log(`서버 시작`);
});
```

<br>

서버 구동

```
node index.js
```

<br><br>

### 🌊 테스트 스크립트 작성

<br>

테스트 스크립트를 작성한다. 

``` yaml
# login_test.yaml
config:
  target: "http://localhost:8080" 
  phases:
    # 1단계: /login API 부하 테스트 (60초 동안 초당 10명)
    - duration: 60
      arrivalRate: 10
      name: "Phase 1: Direct Login Test"
      scenario: "User Login Scenario"

      
    # 2단계: /login-delay API 부하 테스트 (60초 동안 초당 10명)
    # 첫 번째 단계가 끝난 후 이어서 실행됨
    - duration: 60
      arrivalRate: 10
      name: "Phase 2: Delayed Login Test"
      scenario: "User Login with Delay Scenario"


  # users.csv 파일에서 테스트 데이터를 읽어옴
  payload:
    path: "./users.csv"
    fields:
      - "id"
      - "pw"
    skipHeader: true
    loop: true 

scenarios:
  # 첫 번째 시나리오: /login API 호출
  - name: "User Login Scenario"
    flow:
      - post:
          url: "/login"
          json:
            id: "{{ id }}"
            pw: "{{ pw }}"
          capture:
            json: "$.message"
            as: "login_response_message"

  # 두 번째 시나리오: /login-delay API 호출
  # 이 시나리오를 사용하는 가상 사용자가 Phase 2에서 생성됨
  - name: "User Login with Delay Scenario"
    flow:
      - post:
          url: "/login-delay"
          json:
            id: "{{ id }}"
            pw: "{{ pw }}"
          capture:
            json: "$.message"
            as: "login_delay_response_message"
```

<br>

스크립트에서 각 구문이 어떤 역할인지 살펴보자. 
- `target`: Artillery가 부하를 보낼 목표 서버의 기본 URL
- `phases`: 테스트를 단계별로 어떻게 진행할지 정의
- `duration`: 현재 단계가 지속될 시간을 초 단위로 정의
- `arrivalRate`: 초당 몇 명의 새로운 가상 사용자(VU)를 생성할지 정의
- `scenarios`: 각각의 시나리오 흐름을 정의(미입력 시 랜덤으로 API 호출)

<br>

`path: "./users.csv"` 구문을 통해 파라미터로 사용할 CSV 데이터를 임포트 할 수 있다.

```
// users.csv
id,pw
user1,pass1
user2,pass2
user3,pass3
user4,pass4
user5,pass5
```

<br>

따라서 위 스크립트는 첫 번째 단계에서 60초 동안 초당 10명의 가상 사용자를 생성하여 `/login` API에 부하를 가하고, 두 번째 단계에서 60초 동안 초당 10명의 가상 사용자를 생성하여 3초 지연이 있는 `/login-delay` API에 부하를 가한다. 

<br><br>

### 🌊 테스트 진행

아래와 같은 커맨드로 테스트를 진행할 수 있다. 
시험 결과에 대한 로우 데이터를 `login_test_report.json`으로 출력한다. 

```
artillery run login_test.yaml --output login_test_report.json 
```

<br>

정상이라면 아래와 같이 출력되다가 테스트 종료 시 로그 파일 생성을 알려준다.

```
Started phase 0 (Phase 1: Direct Login Test), duration: 60s @ 12:47:52(+0900) 2025-09-02
Report @ 12:48:02(+0900) 2025-09-02
Elapsed time: 10 seconds
  Scenarios launched:  99
  Scenarios completed: 92
  Requests completed:  92
  Mean response/sec: 10.01
  Response time (msec):
    min: 0
    max: 1040
    median: 999.5
    p95: 1012.8
    p99: 1029.9
  Codes:
    200: 92

... 

Log file: login_test_report.json
```

<br>

로우 데이터를 확인해도 테스트 결과를 이해하기 어려우니 시각화해서 확인해 보자. 

```
artillery report login_test_report.json
```

![](/assets/tech/server-stress-test-with-artillery/image1.png)

`Test duration`을 통해 130초 동안 테스트가 진행되었다는 것을 알 수 있고, 1200번의 요청에 대한 응답이 모두 HTTP 200으로 도착하였다. 차트에서는 아래 정보를 확인할 수 있다. 

- `Min`: 테스트 기간 동안 발생한 모든 요청 중 서버가 가장 빨리 응답한 시간
- `Max`: 테스트 기간 동안 발생한 모든 요청 중 서버가 가장 느리게 응답한 시간
- `Median`: 모든 응답 시간을 오름차순으로 정렬했을 때, 정확히 가운데에 위치하는 값
- `P95 (95th Percentile)`: 모든 응답 시간을 오름차순으로 정렬했을 때, 하위 95%에 해당하는 요청들이 이 시간 안에 응답을 완료했다는 의미
- `P99 (99th Percentile)`: 모든 응답 시간을 오름차순으로 정렬했을 때, 하위 99%에 해당하는 요청들이 이 시간 안에 응답을 완료했음을 의미

<br>

![](/assets/tech/server-stress-test-with-artillery/image2.png)

- `Latency At Intervals`: 테스트 진행하는 동안 요청에 대한 `Latency` 
- `Concurrent users`: 특정 시점에 서버에 요청을 보내고 있는 가상 사용자의 수(서버에 요청되었지만 아직 응답을 받지 못한 가상 사용자 수)
- `Mean RPS`: 테스트 기간 동안 1초에 평균적으로 몇 개의 요청이 서버에 성공적으로 전달되었는지를 의미

<br><br><br><br><br>

# 📌 테스트를 통한 성능 개선 예시

<hr>

## ✨ 캐시로 Latency 최적화하기

극단적인 예시로 사용자 ID를 입력받아서 해시를 5만 번 돌린 후 반환하는 API가 있다고 치자. 

``` javascript
// index.js
const crypto = require('crypto');


app.post('/login-hash', (req, res) => {
  const { id } = req.body;
  if (!id) return res.status(400).json({ message: 'ID 미입력' });
  
  let hash = id;
  for (let i = 0; i < 50000; i++) hash = crypto.createHash('sha256').update(hash).digest('hex');
  console.log(`해시 결과: ${hash}`);
  res.status(200).json({ processed_id: hash });
});
```

``` yaml
# login_test.yaml
config:
  target: "http://localhost:8080"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Phase: Direct Login Hash Test"
      scenario: "User Login-hash Scenario"


  # users.csv 파일에서 테스트 데이터를 읽어옴
  payload:
    path: "./users.csv"
    fields:
      - "id"
    skipHeader: true
    loop: true 

scenarios:
  - name: "User Login Scenario"
    flow:
      - post:
          url: "/login-hash"
          json:
            id: "{{ id }}"
          capture:
            json: "$.message"
            as: "login_response_message"
```

![](/assets/tech/server-stress-test-with-artillery/image3.png)

시험 결과를 리포트로 출력해 보면 모든 요청이 정상적으로 평균 80 m/s 이내에 응답하였다는 것을 확인할 수 있다. 


<br><br>  

이번에는 해시 반복 횟수를 10만 번으로 늘려보자. 

``` javascript
// index.js
const crypto = require('crypto');


app.post('/login-hash', (req, res) => {
  const { id } = req.body;
  if (!id) return res.status(400).json({ message: 'ID 미입력' });
  
  let hash = id;
  for (let i = 0; i < 100000; i++) hash = crypto.createHash('sha256').update(hash).digest('hex');
  console.log(`해시 결과: ${hash}`);
  res.status(200).json({ processed_id: hash });
});
```

![](/assets/tech/server-stress-test-with-artillery/image4.png)

시험 결과를 리포트로 출력해 보면 `Latency`의 `P99`가 10초 이상이며, 테스트 1분 경과 후에는 응답이 출력되지 않는 것을 확인할 수 있다.  
이는 서버에 부하로 인한 `Time Out`이 발생하였기 때문이다. 

<br><br>

해시를 반복하는 구문이 병목 구간이라는 가설을 발굴했다. 
이제 캐시를 도입하여 이를 개선해 보자. 

``` javascript
// index.js
const hashCache = new Map(); 

app.post('/login-hash', (req, res) => {
  const { id } = req.body;
  if (!id) return res.status(400).json({ message: 'ID 미입력' });

  if (hashCache.has(id)) {
    const cachedHash = hashCache.get(id);
    console.log(`해시 결과: ${cachedHash}`);
    return res.status(200).json({ processed_id: cachedHash });
  }
  
  let hash = id;
  for (let i = 0; i < 100000; i++) hash = crypto.createHash('sha256').update(hash).digest('hex');
  hashCache.set(id, hash);
  console.log(`해시 결과: ${cachedHash}`);
  res.status(200).json({ processed_id: hash });
});
```

![](/assets/tech/server-stress-test-with-artillery/image5.png)

요청받은 이력이 있는 사용자 ID라면 기존에 계산했던 해시 값을 반환하도록 캐시를 도입했다. 
그 결과 `Latency`가 점차 줄어드는 것을 확인할 수 있다. 
(실전에서는 런타임 메모리에 저장하는 방식이 아닌, `Redis` 등 제3자 서비스에 캐시를 구성해야 한다.)

<br><br><br><br><br>

# 📌 References
🔗 [Node.js Artillery](https://node-js.tistory.com/36)<br>
🔗 [Node.js Artillery](https://poleved.tistory.com/217)<br>
🔗 [Artillery(테이블링 테크블로그)](https://techblog.tabling.co.kr/artillery%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%B6%80%ED%95%98-%ED%85%8C%EC%8A%A4%ED%8A%B8-9d1f6bb2c2f5)<br>