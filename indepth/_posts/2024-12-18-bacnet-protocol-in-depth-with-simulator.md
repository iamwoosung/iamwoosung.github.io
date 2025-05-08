---
layout: post
title: 백넷 프로토콜 톺아보기(With 시뮬레이터)
description: >
  백넷 프로토콜 학습, 개발, 운영 경험에 대한 포스팅입니다.
sitemap: true
hide_last_modified: true
---

* toc
{:toc .large-only}

<br><br><br><br><br>

# 📌 Summary
<hr>
나는 빌딩, 팩토리, 공항 등 대형 건물에서 사용하는 재난 모니터링 솔루션 개발 경험이 있다.
건물에 설치되는 승강기, 소방, 조명 등 공종들과 연동하여 실시간 데이터를 관제해야 한다.

<br>

데이터를 받아올 때 대부분의 공종들이 자체 프로토콜(공종 업체의)로 통신을 제공하는데, 
설비 공종은 표준화된 `백넷(BACnet/IP)`이라는 프로토콜로 데이터를 제공한다.

<br>

설비 기기와 연동을 위해 `백넷(BACnet/IP)` 프로토콜을 학습, 개발한 내용과
현장 구축을 하며 발생된 이슈, 그리고 해결 방법에 대해 기록한다. 
(회사에서 작성한 코드라 지적재산권 이슈로 코드는 첨부하지 않는다.)

<br><br><br><br><br>

# 📌 백넷(BACnet)이란 무엇인가
<hr>

## ✨ 백넷(BACnet)이란 

`BACnet` 프로토콜은 아래 약자처럼 건물 자동화를 위한 프로토콜이다.

> "A Data Communication Protocol for **B**uilding **A**utomation and **C**ontrol **Net**works"

<br>

미국 냉동기술 학회(ASHRAE)에서 1995년 제안하여 현재까지 30여 개국에서 표준으로 채택되었다. 
국내 설비 업체들도 `백넷`을 사용하여 범용 프로토콜로 표준화되었다.

<br>

`백넷`은 6가지 네트워크 방식을 지원한다.
- BACnet ARCnet
- BACnet ISO 8802-3 (Ethernet)
- BACnet LonTalk
- BACnet MS/TP (Master-Slave/Token Passing)
- BACnet Point-to-Point (EIA-232)
- BACnet/IP

일반적으로 `TCP` 기반의 `BACnet/IP` 방식만 지원하는데, 
`시리얼` 기반인 `BACnet MS/TP`를 라우팅 처리하여 `BACnet/IP`와 동시 지원하는 곳도 있다.

<br>

https://bacnet.org/ 에서 프로토콜 종류나 개발 예제를 자세히 제공하고 있다.

<br><br><br>

## ✨ 백넷으로 어떤 데이터를 주고받을까
건물에 설치되는 설비 기기는 보일러, 물탱크, 공조기, 냉난방기, 배수관 등 다양하다. 

<br>

아래와 같은 공조기(HVAC)로 예를 들자면, 
공조기 내부에는 환기온도, 외기온도, 습기온도, 급기휀 등의 센서가 있으며 센서 별로 각각의 데이터가 존재한다.

<br>

`백넷`으로 주고받는 데이터는 기기(공조기 등) 단위가 아닌, 
기기 내부에 들어있는 센서(환기온도, 외기온도 등) 단위이다.

![](https://velog.velcdn.com/images/iamwoosung/post/bad01a56-99a3-4820-9ff3-8d16b8e7d186/image.png)


설비 업체는 DDC라 불리는 제어 장치에 공조기 같은 기기들을 물리적으로 한대 이상 연결한다. 
기기는 한 대만 연결될 수도, 몇백 대가 연결될 수도 있다. 설비 업체에서 정의

<br>

DDC는 기기에 포함된 센서의 값을 주기적으로 갱신하며 갖고 있는다.
DDC는 서버 역할이고, 우리는 DDC에 `백넷` 프로토콜로 실시간 데이터를 요청하는 클라이언트이다. 


<br><br><br>

## ✨ 백넷을 사용하는 이유 
지금까지 사이트 구축을 하면서 만난 모든 설비 업체들이 `BACnet/IP` 프로토콜로 데이터를 제공했다. (지멘x, 하니x 등)

<br>

그렇다면, 속도적인 측면에서 `RS 232`, `TCP/IP`가 더 빠를 수 있을 텐데,
왜 대형 건물에서는 `백넷`을 사용하는지 의문이 생길 것이다.

<br>

`백넷`은 오브젝트(Object)를 지향하고, 기기(정확히는 센서)의 정보를 오브젝트 단위로 제공한다.
각 오브젝트에는 속성(Property)을 정의하는 기능이 있는데, **빌딩 제어와 자동화에 맞춤화**되어 있다. 
클라이언트는 속성을 통해 오브젝트의 정보와 상태를 알 수 있다.
- 현재 값이 몇인지, 
- 경보(기기 이상) 상태인지, 
- 어떤 종류인지(Analog, Binary), 
- 어떤 역할인지(Input, Output, Value), 
- 어떤 디바이스에 포함되어 있는지 등등

<br>

프로토콜 내에 포함되는 속성들의 성격이 빌딩에 겨냥되어 있기 때문에, 
설비 업체 입장에서는 표준화된 프로토콜을 사용하여 벨류를 간단하게 전달할 수 있다.
클라이언트 입장에서도 편하다. (프로토콜 문서를 작성, 확인하지 않아도 된다!..)
또한 속성 추가가 자유롭기 때문에 필요에 따라 커스텀 가능하다.

<br><br><br>

## ✨ 주관적인 백넷의 단점

**🍀 암호화되지 않은 프로토콜**

백넷은 기본적으로 패킷이 암호화되어 있지 않아서 와이어 샤크 같은 프로그램으로 패킷을 확인하면 장비명, 위치, 현재 값을 확인하는 것이 가능하다.

이를 악용하여 장비에 마음대로 제어 명령을 요청할 수 있다. 이를 방지하기 위해 폐쇄 망으로 DDC와 제어용 소프트웨어만 통신이 가능하도록 구성하는 것이 일반적이다.

<br>

**🍀 우선순위의 남용**
백넷은 우선순위라는 기능이 있다. 장비보다 낮은 우선순위의 소프트웨어로는 제어가 불가하다.

분명 필요한 기능이다. 대규모 현장이면 제어용 소프트웨어가 다수 개 연동되기 때문이다. 
각 소프트웨어에서 요청한 값을 반영할지 우선권을 설정하여 **의도에 맞는 제어와 다양한 예외 방지**가 가능하기 때문이다. 

(백넷 프로토콜이 아닌 업체에 대한 불만.. 😥)
문제는 거의 모든 장비에 불필요한 우선순위를 설정해 놓는다. 클라이언트 입장에서는 포인트마다 각기 다른 우선순위를 부여해야 하기 때문에 작업량과 기간이 늘어난다. 뿐만 아니라, 사전 공유가 이루어지지 않거나 구축 이후의 수정으로 두번 작업하는 일이 허다하다. 

<br><br><br>

## ✨ 백넷 오브젝트의 종류
[DOC](https://bacnet.org/wp-content/uploads/sites/4/2022/06/The-Language-of-BACnet-1.pdf)를 확인해 보면 기본적으로 18가지 오브젝트를 지원한다고 명시되어 있다. 
여기서 가장 기본적이고 사용이 잦은 오브젝트는 Device, Analog, Binary, Schedule이다.

- Device: 
  - DDC 단위이다.(하나의 DDC는 하나의 Device)
  - 포함된 포트 내에서 **고유한 Device ID**를 가진다.
  - Device 하위에 Analog, Binary, Schedule 등의 오브젝트를 생성한다.
- Analog: 
  - **Float 형식**의 값을 저장한다.
  - 포함된 DDC의 동일 오브젝트는 **고유한 인스턴스 번호**를 가진다.
  - 온습도나 압력, 수위 등의 값을 저장할 때 사용한다.
  - 특정 값 이상 또는 이하일 때 경보 상태로 변경할 수 있다.
  - **Input(AI), Output(AO), Value(AV)로 구분**된다.
- Binary: 
  - **Bool 형식**의 값을 저장한다.
  - 포함된 DDC 내에서 **고유한 인스턴스 번호**를 가진다.
  - 환기휀 동작 여부, DDC 연결 상태 등 On/Off 값을 저장할 때 사용한다.
  - 특정 값 이상 또는 이하일 때 경보 상태로 변경할 수 있다.
  - **Input(DI), Output(DO), Value(DV)로 구분**된다.
  - Digital로도 불린다.
- Schedule:
  - 건물 자동화를 위한 기능으로 **원하는 시간에 센서를 자동 제어**하는 기능이다.
  - 클라이언트에서는 Read만 가능하고, 설비 업체에서 정의한다.

<br>

Analog, Binary 오브젝트는 Input, Output, Value로 역할을 구분한다.

- Input: 센서에서 측정되는 값을 저장하는 오브젝트이다.
- Output: 제어 명령을 받기 위한 오브젝트이다. (클라이언트에게 제어 명령을 받는다.)
- Value: 센서나 시스템의 현재 값을 표출하는 오브젝트이다.(Input, Output의 영향을 받는다.)

<br><br><br><br><br>

# 📌 백넷의 통신 프로세스
<hr>
  
  
0. `백넷` 서버가 구동되면 설정한 IP 대역과 포트에 고유 디바이스 번호로 바인딩 된다.
   - 여러 IP가 설정되어 있을 경우 일반적으로 하나의 IP 대역에만 바인딩 된다.
   - 47808번 포트가 기본이다.
   <br>
1. 클라이언트가 구동되면 자신의 IP 대역에 `Who-Is` 패킷을 브로드캐스트로 전송하며 `백넷` 서버를 검색한다.
   <br>
2. `Who-Is` 신호를 수신 받은 백넷 서버는 `I-am` 패킷으로 응답한다.
   - 해당 패킷에 디바이스 정보에 대한 내용이 응답으로 포함될 수 있다.
   - 해당 패킷에 하위 오브젝트 리스트는 응답에 포함되지 않는다. 
   <br>
3. 클라이언트에서 필요한 `백넷` 서버에만 Request 하여 오브젝트의 현재 값 등을 가져온다.
   - 하나의 패킷에 하나의 오브젝트만을 받을 수 있다.
   - 오브젝트 리스트는 하나의 패킷으로 받을 수 있다.
 
 
<br>

![](https://velog.velcdn.com/images/iamwoosung/post/6c4d376c-a696-4b40-952e-3ae00974d146/image.png)



<br><br><br><br><br>

# 📌 시뮬레이터로 이해하는 백넷 구성
<hr>
텍스트 설명만으로는 개념을 이해하는 것이 어렵다.
시뮬레이터를 이용하여 `백넷` 서버와 클라이언트를 구성해 보자.
(온보딩 목적으로 구성했던 환경인데, 이해를 돕는데 가장 적합한 거 같다.)


<br><br><br>

## ✨ 백넷 서버 시뮬레이터 프로그램
[SCADA Engine](https://www.scadaengine.com/downloads.php?product=bacnet_simulator)에서 제공하는 30일 트라이얼 버전의 프로그램을 사용했다.
~~(라이센스 파일 삭제하면 트라이얼 기간 리셋 가능,,)~~

<br>

설치한 후 프로그램을 실행하면 기본으로 구성된 서버가 존재한다.

![](https://velog.velcdn.com/images/iamwoosung/post/a9d06421-9d76-4994-aeff-4a6c29646c3f/image.png)

<br><br><br>

서버 > 우클릭 > Add Device를 선택하여 디바이스를 생성해 보자

![](https://velog.velcdn.com/images/iamwoosung/post/fd8fcd67-387b-4328-a701-db1e6f3943d8/image.png)

<br><br><br>

아래와 같이 디바이스가 추가되는데, 현장에서 구축되는 하나의 DDC라고 이해하면 된다.
** 정의한 디바이스 명과 ID로 서버에 바인딩된다. **
디바이스 > 우클릭 > Add Object로 하위에 오브젝트를 생성해 보자
 
![](https://velog.velcdn.com/images/iamwoosung/post/db115e56-8dd5-4cfa-84ce-33a9b703e68c/image.png)

<br><br><br>

다양한 오브젝트를 지원하는데 테스트 용으로 AI, AO, AV, DI, DO, DV를 만들어보자.

![](https://velog.velcdn.com/images/iamwoosung/post/52fe7235-5881-4725-a68f-f7ece545deb4/image.png)

<br><br><br>

생성된 여러 개의 태그 중 하나를 선택하면, 
어떤 속성이 설정되어 있고 각 속성별로 어떤 값을 갖고 있는지 확인할 수 있다.

PresentValue가 현재 값을 나타내는 속성인데, 변경할 수 있다.
(Simulator Settings에서 랜덤, 증감, 감소 값으로 설정할 수 있다.)

![](https://velog.velcdn.com/images/iamwoosung/post/d48fea81-ec1c-4806-9ea8-e8e2d21cc1bb/image.png)

<br><br><br>

## ✨ 백넷 클라이언트 프로그램

서버 시뮬레이터를 설정했다면 백넷 클라이언트로 데이터를 받아올 차례이다. 
[sourceforge](https://sourceforge.net/projects/yetanotherbacnetexplorer/)에 있는 Yabe 프로그램이 나름 테스트 툴로 상용화되어 있다.

<br>

설치하여 실행한 이후, 우측 상단의 + 버튼을 클릭하면 채널 설정 란이 팝업 된다.
`BACnet/IP`를 사용할 거기 때문에 해당 정보만 입력해 주면 된다.
- Port: `백넷`은 47808을 디폴트 포트로 사용한다. (47808 == BAC0)
- Local endpoint: 피시에 IP가 다수 개 등록되어 있을 때, 검색할 대상 IP 대역이다.

![](https://velog.velcdn.com/images/iamwoosung/post/7593539a-f301-42a7-98a3-5a4faa5ce8e7/image.png)

<br><br><br>

Start 하면 해당 포트와 IP 대역에서 검색된 백넷 디바이스들이 출력된다.
(Start 시 클라이언트가 전송한 `Who-Is` 패킷에 `I-am` 패킷으로 응답한 디바이스 리스트)

![](https://velog.velcdn.com/images/iamwoosung/post/346cc2f4-d776-415c-9630-b0c491fe4b3d/image.png)

<br><br><br>

디바이스 하나를 선택하면 하위에 어떤 오브젝트들이 포함되어 있는지 표출된다.

![](https://velog.velcdn.com/images/iamwoosung/post/e8326482-5d82-4f9c-affd-afbbd009d200/image.png)

<br><br><br>

드래그 & 드롭한 오브젝트에 한하여, Polling 방식으로 값을 지속 갱신한다.
우측에는 서버에서 설정된 상세 속성 정보와 밸류를 확인할 수 있다.

![](https://velog.velcdn.com/images/iamwoosung/post/c04df3fe-b181-4554-ab55-5c987a752c45/image.png)


<br><br><br><br><br>

# 📌 Polling의 문제점과 COV 도입을 통한 개선
<hr>

첫 사이트에 백넷 클라이언트를 구축했을 때 초반부에는 원활하게 동작했지만, 
구동 시간이 증가할수록 서버로부터 응답 패킷이 느리게 도착했다.

<br>

원인은 DDC를 관리하는 백넷 서버의 과부하였는데, 서버가 너무 많은 패킷을 수신하며 부하가 발생했다.
우리 회사의 솔루션 뿐 아니라, 다른 업체에서도 백넷 서버에 데이터를 요청하고 있었다. 
(우리 솔루션이 요청하는 패킷 수가 월등하게 많아서 개선이 필요했다.)

<br><br><br>

## ✨ Polling의 문제점

개발한 백넷 클라이언트 기능 및 사양은 1,000 ms에 한 번씩, 
900여 개 오브젝트를 Polling 하여 실시간 값과 이상 유무를 관제하는 것이었다. 

<br>

900여 개 오브젝트를 Polling 방식으로 갱신하려면, 
1,000 ms를 단위로 900여 개의 패킷이 송/수신된다는 것과 같다.

<br>

서버의 부하를 줄이기 위해 Read 주기를 늘릴 수 있었지만, 
실제 센서의 값이 변경되었을 때, 우리 솔루션에 관제되기까지 딜레이가 발생되어 실시간성을 보장할 수 없었다.

<br><br><br>

## ✨ COV의 도입

도큐먼트와 레퍼런스를 검색하다가 COV에 대한 내용을 발견했다.

<br>

COV(Change Of Value)는 클라이언트가 서버에게 모니터링할 오브젝트 리스트를 넘겨주면, 
서버에서 해당 오브젝트가 변경되었을 때 클라이언트에게 패킷을 보내주는 개념이다.

<br>

COV 기능은 변경된 오브젝트만 알려준다. 
클라이언트가 구동된 이후 변경되는 오브젝트가 없다면 값을 알 수 없기 때문에, 
첫 1회는 Polling 방식으로 값을 갱신했다.

<br>

구현을 하려고 보니 COV 지속 시간을 서버에게 알려줘야 했다. (반영구인 줄 알았는데)
Polling 이후, COV를 1시간마다 재신청하게끔 타이머 스레드를 구성했고, 
서버에게 변경된 오브젝트만 받아서 갱신하는 방식으로 개선하였다.

<br>

**🍀 AS-IS**

![](https://velog.velcdn.com/images/iamwoosung/post/fb973cf4-00e6-4b5b-a8b7-bf90977243e5/image.jpg)

<br>

**🍀 TO-BE**

![](https://velog.velcdn.com/images/iamwoosung/post/3afb2ff1-bc10-4393-a3ff-75a6430154bb/image.jpg)

	
<br><br><br>

## ✨ COV를 옵션 처리해야 하는 이유
COV를 지원하지 않는 설비 업체가 있다. (이유는 모르겠지만,,)
COV의 지원 여부를 확인하는 방법은 간단하다.
COV 요청 패킷에 대한 응답 패킷이 Reject로 수신되는지 확인하면 된다. 
(업체에게 문의하는 게 제일 빠르고 정확하긴 함)
![](https://velog.velcdn.com/images/iamwoosung/post/9bc9c896-68f6-4b8a-ac5e-b9c71f905c9a/image.png)

<br> 

또한 시간 단위로 COV 패킷 제한이 있는 업체도 있다. 
제한 범위를 넘어서면 오브젝트가 변경되더라도 COV 패킷이 송신되지 않는다.

<br> 

위와 같은 경우에는, 프로세스나 스레드를 다수 개로 나누더라도 Polling 방식을 채택해야 한다.

<br><br><br>

## ✨ COV 개발 레퍼런스

Yabe 개발에 사용된 `C#` 오픈 소스에도 해당 기능이 포함되어 있다. 

``` cs
// System.IO.BACnet.BacnetClient

public bool SubscribeCOVPropertyMultiple(BacnetAddress adr, BACnetSubscribeCOVPropertyMultiple s_cov_multiple, byte invoke_id = 0);
public bool SubscribeCOVRequest(BacnetAddress adr, BacnetObjectId object_id, uint subscribe_id, bool cancel, bool issue_confirmed_notifications, uint lifetime, byte invoke_id = 0);
```

<br> 

https://github.com/ela-compil/BACnet.Examples 
위 오픈 소스에도 `BACnet` Handler와 Function이 상당히 많이 구현되어 있다. 👍

<br><br><br><br><br>

# 📌 백넷에 대한 주관적인 생각
<hr>


구축 경험치가 좀 쌓이고 돌아보니 데이터 송/수신 방식은 편하지만, 결코 빠르지 않다. 
속도적인 측면은 개발 시 병렬 처리에 신경 써줘야 운영이 가능하다. 
그리고 하나의 DDC에 오브젝트가 500여 개 이상이면 Polling 방식은 적합하지 않은 거 같다.

<br>

하지만 장점도 명확히 존재한다. 표준화된 프로토콜인 만큼 `백넷`을 사용하는 업체들이 많다.
따라서 초기에 한 번만 잘 개발해 놓는다면 타 현장에 쉽게 구축할 수 있고, 
PM이나 회사 입장에서 장기적으로 리소스도 절감된다.

<br>

`백넷` 프로토콜이 베이스가 되는 앱은 소프트웨어 인증을 받는 것을 추천한다.
공공기관, 대기업에 수주 시 GS(Good Software)나 BTL(Building Automation and Control Network) 인증서를 요구하는 곳이 많았다.

<br>

**결론** 잘 개발된 백넷 클라이언트를 사용하자.

<br><br><br><br><br>

# 📌 References
<hr>

🔗 [BACnet](https://bacnet.org/wp-content/uploads/sites/4/2022/06/The-Language-of-BACnet-1.pdf)<br>
🔗 [BACnet 기능](https://bacnet.org/wp-content/uploads/sites/4/2022/06/How-to-Specify-BACnet-Based-Systems-1.pdf)<br>
