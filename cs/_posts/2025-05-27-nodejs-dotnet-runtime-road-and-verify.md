---
layout: post
title: Node.js에서 .NET 런타임 로드하고 검증하기
description: >
  Node.js에서 edge-js 라이브러리를 통한 .NET 런타임 로드 및 검증과 C# 코드 호출 방법에 대한 포스팅입니다.
sitemap: true
hide_last_modified: true
---

* toc
{:toc .large-only}

<br><br><br><br><br>

# 📌 Overview

<hr>

`Node.js` 웹 서버를 개발하던 중 `C#` 코드 호출이 필요한 기능이 있었다. <br>
`C#` 모듈의 `HTTP Listner` 오픈이나 `IPC` 통신으로 `Node.js`에게 데이터 전달을 고려했지만 비효율적이라 망설이던 중 `.NET` 런타임을 호환할 수 있게 도와주는 `edge-js` 라이브러리를 발견하였다. 

<br>

별 기대 없이 사용성 검토를 진행했는데 예상외로 간단한 적용 방법과 빠른 처리 속도를 보여주었다. (쓸데없을 수도 있지만) 내부 동작 방식이나 `.NET` 런타임의 로드 시점이 궁금하여 나름대로 파보았고 그 과정에서 학습하게 된 내용을 기록한다. 

<br><br><br><br><br>

# 📌 .NET 런타임이란?

<hr>

본론에 들어가기 앞서, `.NET` 런타임과 `Node.js`의 네이티브 애드온을 이해하는 것이 좋을 거 같다.

<br>

`.NET`(닷넷)은 `C#` 프로그래밍 언어로 작성된 코드를 실행시켜주는 플랫폼이다. <br>
`C#`으로 작성된 코드를 컴파일하면 중간 언어인 `IL`(Intermediate Language) 코드로 변환되고, `IL` 코드를 실행시키려면 `.NET` 런타임이 필요하다. 

<br>

조금 더 자세히 설명하자면, 우리는 `IDE`를 통해 `C#` 코드를 작성하고 컴파일한다. `Visual Studio`의 경우에는 컴파일 성공 시 기본적으로 Debug 또는 Release 디렉터리에 .exe 확장자의 바이너리 실행 파일이 생성된다. 

바이너리 파일은 `IL` 코드로 구성되어 있다. 컴파일 과정에서 `C#` 코드가 `IL` 코드로 변환된 것이며 해당 바이너리 파일을 실행하기 위해서는 `.NET` 런타임(SDK)이 설치되어 있어야 한다. 

컴파일한 환경에는 당연히 `.NET` 런타임이 설치되어 있기에 문제없이 실행될 테지만 `.NET` 런타임이 미설치된 환경에서는 해당 바이너리를 실행할 수 없다는 의미이다. 

<br>

P.S. 배포 시 설정으로 바이너리 파일에 `.NET` 런타임을 포함시킬 수 있다고 하는데 해본 경험이 없어서 호환성 이슈나 윈도우가 아닌 `OS`에서 정상 동작할지는 의문이다. 🤔 <br>
(기회가 된다면 확인해 볼 예정,,) 


<br><br><br><br><br>

# 📌 네이티브 애드온이란?

<hr>

`Node.js`는 `C++`로 작성된 `V8` 자바스크립트 엔진과 코어 위에서 구동된다. 설계 단에서부터 이미 `C++`과 친숙하게 구성되어 있기 때문에 `C++`로 작성한 네이티브 코드를 `Node.js`가 이해할 수 있게 변환해 주기만 `require()` 함수를 통해 손쉽게 호출하고 사용할 수 있다. 이러한 기능을 **네이티브 애드온(Native Addon)** 이라고 한다.

<br>

네이티브 애드온의 로직은 아래와 같다. (깊게 파면 끝이 없는 개념이라 개략적으로만 설명)

1. `C++` 코드를 작성한다.
2. `C++` 네이티브 코드를 `Node.js`가 이해할 수 있게 컴파일한다.
   - `C++` 컴파일러 필요
   - 일반적으로 `node-gyp` 라이브러리로 변환
   - 변환 성공 시 `.node` 확장자로 파일 생성
3. 컴파일된 `.node` 파일을 `require()` 한다. 
4. `C++` 함수의 리턴 값은 `V8/N-API`를 통해 `Javascript` 형태로 마샬링되어 반환된다.

<br>

주요는 `V8/Node-API`라는 표준 인터페이스를 통해 `Node.js` 환경에서 `C++` 코드를 호출하고 데이터를 마샬링하는 것이다.

<br><br><br><br><br>

# 📌 edge-js란?

<hr>

위 내용으로 네이티브 애드온이 어떤 역할을 수행하는지, `Node.js`에서 어떤 원리로 `C++` 네이티브 코드를 호출하는지 개략적으로 이해하였을 것이다. 

<br>

그럼 만약 `C++` 코드가 아닌, `C#` 코드를 호출하고 싶다면 어떻게 할 수 있을까? <br>
네이티브 애드온을 활용한 `edge-js` 라이브러리를 사용하면 해결할 수 있다. 

<br>

`C#`은 네이티브 코드가 아닌 관리 코드 형식인데, 실행을 위해서는 컴파일과 메모리 관리 역할의 `.NET CLR`이 필요하다. **`edge-js`는 `C++` 코드를 통해 `Node.js`가 구동되는 프로세스 내부에 `.NET CLR`을 로드시킨다.**

<br>

쉽게 정리하자면 `Node.js`와 `C++`은 네이티브 애드온을 통해 기본적으로 호환이 가능하다. `Node.js`와 `C#`은 `edge-js`가 중간다리 역할을 해줌으로써 호환이 가능하다.

<br><br><br><br><br>

# 📌 edge-js 사용 방법

<hr>

## ✨ C# DLL 파일 생성하기

먼저 `Node.js`에서 호출할 `C#` 코드를 확장자 .dll 파일로 생성해야 한다.


간단하게 두 개의 입력값을 더해주는 코드를 작성했다.

``` cs
using System.Threading.Tasks;

namespace SampleCode
{
	public class Startup
	{
		public async Task<object> Invoke(dynamic input)
		{
			return ((int) input.input1 + (int) input.input2);
		}
	}
}
```

<br> 

위 코드를 .dll 파일로 컴파일 해줘야 하는데, `.NET` 프레임워크가 설치되어 있어야 한다. <br>
명령 프롬프트에서 csc 파일을 실행하며 인자 값으로 컴파일할 `C#` 파일을 타겟으로 넘겨준다.<br>
(csc.exe 파일의 위치는 피시 환경이나 프레임워크 버전마다 상이할 수 있다.)

```
C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe /target:library SampleCode.cs
```

<br>

다소 오류처럼 보이는 문구가 출력되는데, 잠시 기다리면 .dll 파일이 생성된다.
```
Microsoft (R) Visual C# Compiler version 4.8.9232.0
for C# 5
Copyright (C) Microsoft Corporation. All rights reserved.

This compiler is provided as part of the Microsoft (R) .NET Framework, but only supports language versions up to C# 5, which is no longer the latest version. For compilers that support newer versions of the C# programming language, see http://go.microsoft.com/fwlink/?LinkID=533240

SampleCode.cs(7,29): warning CS1998: 이 비동기 메서드에는 'await' 연산자가 없으며 메서드가 동시에 실행됩니다. 'await'
        연산자를 사용하여 비블로킹 API 호출을 대기하거나 'await Task.Run(...)'을 사용하여 백그라운드 스레드에서 CPU
        바인딩된 작업을 수행하십시오.
```

![](/assets/cs/nodejs-dotnet-runtime-road-and-verify/image4.jpg)


<br><br><br>

## ✨ edge-js로 DLL 파일 호출하기

`C#` 코드를 .dll 파일로 생성했다면 이제 `Node.js`에서 호출해야 한다. 

먼저 `edge-js`를 설치하자

``` 
npm i edge-js
```

<br>

아래와 같이 매우매우 간단하게 작성하고 실행해보면 40이 출력된다.

``` javascript
const edge = require("edge-js");
const add = edge.func("SampleCode.dll");

add({ input1: 10, input2: 30 }, function (error, result) {
  if (error) throw error;
  console.log(result);
});
```

<br><br><br>

## ✨ edge-js 활용해서 공유 메모리 액세스하기

사실 `edge-js`를 검토하게 된 이유가 공유 메모리 액세스였다. <br>
`Node.js`에서 공유 메모리의 값을 읽어야 했지만 메모리 맵 단까지 접근하는 게 불가하다고 판단했고, 공유 메모리를 생성한 `C#`이나 `C++` 언어에서 직접 메모리를 읽고 `Node.js`에게 전달하기 위하여 `edge-js`를 검토하게 된 것이다.

(결론적으로는 다른 방법이 채택되었지만 간만에 재미있게 배우면서 검토한 거 같다. 👍)

<br>

아래는 `Node.js`에서 `edge-js`를 통한 `C#` 공유 메모리 맵 생성, 데이터 읽기 예제이다.

<br>

공유 메모리 맵 생성 예제
``` cs
// CreateOrOpenSHM.cs

using System.Threading.Tasks;
using System.IO.MemoryMappedFiles;

namespace CreateOrOpenSHM
{
	public class Startup
	{
        public async Task<object> Invoke(object memoryMapName)
        {
            try
            {
                SHMV v;
                v.value1 = 0;
                v.value2 = 0;
                v.value3 = 0;
                v.value4 = 0;
                v.value5 = 0;

                MemoryMappedFile mmf = MemoryMappedFile.CreateOrOpen((string) memoryMapName, 100000);
                var accessor = mmf.CreateViewAccessor();
                accessor.Write<SHMV>(0, ref v);

                return true;
            }
            catch
            {
                return false;
            }
        }

    }

    struct SHMV
    {
        public int value1;
        public int value2;
        public int value3;
        public int value4;
        public int value5;
    }
}
```
``` javascript
const edge = require('edge-js');
const createOrOpenSHM = edge.func('./CreateOrOpenSHM.dll');

// data는 생성할 메모리 맵 명칭
createOrOpenSHM(data, function(error, result) {
  if (error) {
    console.log("[Fail] 공유 메모리 맵 생성 실패, " + error);
    throw error;
  }
  console.log(result);
});
```


<br>

공유 메모리 읽기 예제
``` cs
// GetSHMDataValues.cs
using System.Threading.Tasks;
using System.IO.MemoryMappedFiles;

namespace GetSHMDataValues
{
	public class Startup
	{
        public async Task<object> Invoke(object memoryMapName)
        {
            try
            {
                SHMV v;

                MemoryMappedFile mmf = MemoryMappedFile.OpenExisting((string) memoryMapName);
                MemoryMappedViewStream stream = mmf.CreateViewStream(0, 12);
                var accessor = mmf.CreateViewAccessor();
                accessor.Read<SHMV>(0, out v);

                return "{ \"value1\": " + v.value1 + ", \"value2\": " + v.value2 + ", \"value3\": " + v.value3 + ", \"value4\": " + v.value4 + ", \"value5\": " + v.value5 + " }";

            }
            catch
            {
                return false;
            }
        }

    }

    struct SHMV
    {
        public int value1;
        public int value2;
        public int value3;
        public int value4;
        public int value5;
    }
}
```
``` javascript 
getSHMDataValues(data, function(error, result) {
  if (error) { 
      console.log("[Fail] 메모리 데이터 값 조회 실패, " + error);
      throw error;
  }
  console.log("[Success] 메모리 맵 데이터 값 조회 성공")
  console.log(result);
});
```

<br><br><br><br><br>

# 📌 .NET CLR 로드 검증하기

<hr>

위에서 "**`edge-js`는 `Node.js` 프로세스에 `.NET CLR`을 로드시킨다**" 라고 설명했었는데, `.NET` 런타임이 정말로 로드되는 것인지 검증하고 싶었고 추가로 로드 시점에 대한 의문점이 생겼다. 

<br>

`.NET CLR`의 로드 시점을 두 가지 방식으로 추론하였다.

🍀 `edge-js`가 `require()` 된 순간부터 해당 프로세스가 종료될 때까지 `.NET CLR`이 로드되어 있는 것일까? <br> 
🍀 `C#` 코드의 호출이 시작될 때 `.NET CLR` 로드했다가 호출이 종료되면 로드를 해제하는 것일까? 

<br>

내 눈으로 직접 확인하고 싶어서 위 내용을 검증해 보기로 하였다. 

<br><br><br>

## ✨ WinDbg로 디버깅하기

`.NET CLR`이 로드되는 것을 어떻게 검증할지 고민하였다. `.NET` 런타임이 구동된다고 해서 프로세스로 검색되는 것이 아니기 때문에 결국 `Node.js` 프로세스 내부를 확인할 수밖에 없었다. 

(프로세스 내부를 디버깅해봐도 유의미한 결과가 나올지 장담할 수 없었지만 마땅히 생각나는 방법이 없어 일단은 해보기로 했다.)

<br><br><br>

**🍀 express 서버**

우선 `.NET CLR`이 로드되지 않는 프로세스를 확인하기 위해 일반적인 express 서버를 구동해보았다.
```
npm i express
```
``` javascript
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => {
  res.send('Hello World!')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})
```

<br>

구동된 `Node.js` 프로세스를 WinDbg에서 Attach 해보면 어떤 파일들을 참조하는지 확인할 수 있다.

![](/assets/cs/nodejs-dotnet-runtime-road-and-verify/image1.jpg)
![](/assets/cs/nodejs-dotnet-runtime-road-and-verify/image2.jpg)

<br> 

어떤 역할의 파일들인지 정확히 알 수는 없으나, `.NET`과 관련한 파일은 로드되지 않은 것을 확인했다.

<br><br><br>

**🍀 edge-js가 포함된 express 서버**

이번엔 `edge-js` `require()` 구문을 추가하고 express 서버를 구동해보았다.
```
npm i express edge-js
```
``` javascript
const edge = require('edge-js')
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => {
  res.send('Hello World!')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})
```

<br>

동일하게 WinDbg에서 Attach 해보면 `.NET`과 관련한 파일들을 확인할 수 있다. <br>  `edge-js` 미포함 시에는 보이지 않던 clr 등의 파일들이 `.NET` 런타임과 관련한 파일이다. 

![](/assets/cs/nodejs-dotnet-runtime-road-and-verify/image3.jpg)

<br>

`edge-js`가 `require()` 되는 시점부터 프로세스 종료 시까지 `.NET` 런타임은 로드된 상태라는 것을 검증할 수 있었다. 😎

<br><br><br><br><br>

# 📌 References

<hr>

🔗 [edge-js 예제1](https://github.com/dealenx/edge-js-example-dll)<br>
🔗 [edge-js 예제2](https://medium.com/@contact_42994/nodejs-edged-with-c-1e9164c1f3eb)<br>
🔗 [WinDbg 사용](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/debugger/getting-started-with-windbg)<br>