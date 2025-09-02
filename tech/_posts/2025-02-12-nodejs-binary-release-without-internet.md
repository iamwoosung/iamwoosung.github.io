---
layout: post
title: Node.js를 폐쇄망에서 바이너리로 릴리즈하기
description: >
  pkg 모듈을 통해 폐쇄망에서 Node.js를 바이너리(.exe) 파일로 릴리즈하였고, 이를 위한 의존성 파일 및 라이브러리 설정 방법에 대한 포스팅입니다.
sitemap: true
hide_last_modified: true
---

* toc
{:toc .large-only}

<br><br><br><br><br>

# 📌 Overview

<hr>

이전에 SI 개발 및 운영 업무를 수행하던 중 구축 현장에서 `Node.js` 앱을 `핫픽스` 한 적이 있었다. <br>
(원래는 사무실 배포 후 현장 반영이 원칙인데 현장과 사무실이 너무 멀었고 소스 코드도 가지고 있었기에,,)

<br>

`Release` 환경은 폐쇄망이었는데 소스 코드에 `node_moules`는 미포함되어 있었다.<br> 
다행히 프록시가 마련되어 있어 `npm`을 통해 다운로드했는데 패키징 과정에서 `pkg` 모듈의 에러가 발생했다.<br> 
원인은 `pkg` 모듈의 일부 설정 파일이 방화벽에 막혀서 누락된 것이었다.. 😥 

<br>

수많은 삽질 끝에 `pkg` 디펜던시를 수동으로 설정하여 `Release` 할 수 있었고 해당 내용을 기록한다. 

<br><br><br><br><br>

# 📌 개발 환경과 라이브러리

<hr>

`Node.js`는 `Docker`나 `클라우드 VM 인스턴스`를 통해 배포하는 것이 일반적이다. (주관적인 의견입니다) <br>
하지만 해당 현장은 바이너리(.exe) 형식으로 `Release` 해야 하며 `Node.js` 런타임이 설치되지 않은 윈도우 서버 `OS`에서 구동되는 것이 클라이언트의 요청사항이었다. 

<br><br><br>

## ✨ 개발 및 구축 환경

> **🍀 개발 환경**<br>
OS: Windows 10<br>
IDE: Visual Studio Code<br>
Node.js Version: 18.13.0<br>
NPM Version: 8.19.3<br>
Network: 폐쇄망<br>

<br>

> **🍀 구축 환경**<br>
OS: Windows Server OS<br>
Program Release Type: Windows Service<br>
Node.js Version: 미설치<br>
Network: 폐쇄망<br>

<br><br><br>

## ✨ 라이브러리 검토

클라이언트의 요구 사항을 수용하기 위해 검토한 라이브러리는 `pkg`이다. <br>
[npm](https://www.npmjs.com/package/pkg)에 명시되어 있는 내용에 따르면 배포된 실행파일은 `Node.js` 미설치 환경에서도 실행 가능하다. <br>
개발 시 사용된 `node_modules` 내용들이 같이 패키징 되기 때문이다.

> This command line interface enables you to package your Node.js project into an executable that can be run even on devices without Node.js installed.


<br><br><br><br><br>

# 📌 pkg 라이브러리

<hr>

## ✨ pkg 사용 방법(희망편)

개발 환경과 네트워크 방화벽이 정상적인 경우에는 아래와 같이 `pkg` 라이브러리를 사용할 수 있다. 

<br>

#### 🍀 설치

``` 
npm i pkg
```

<br>

#### 🍀 코드 작성(express 예제)

``` javascript
// app.js
const express = require('express');
const app = express();

app.get('/test', (req, res) => {
    res.json({
        result: 'Success',
    });
});

app.listen(3000, () => {
    console.log(`server is listening at localhost:${ 3000 }`);
});
```

<br>

#### 🍀 package.json 설정

``` json
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1", 
    "build": "pkg ."
  },
  "bin": "./app.js",
  "pkg":{
    "scripts": [
      "./app.js"
    ],
    "targets": [
      "node18-win-x64"
    ], 
    "outputPath": "dist"
  }, 
```

`package.json` 설정값은 아래와 같은 의미를 가진다.
- `build : pkg .`: npm run build 입력 시 패키징 모듈이 실행
- `bin : ./app.js`: .exe 파일 실행 시 app.js가 구동되도록 설정
- `scripts: [./app.js]`: 패키징에 포함할 주요 파일
- `targets: [node18-win-x64]`: 패키징 환경(Node.js 버전, OS, 비트)
- `outputPath: dist`: 결과물을 출력할 디렉터리 위치
- `assets: [views/**/*]`: 위 예제에서는 사용하지 않았지만 Vue, React 등 프론트 결과물을 views 디렉터리 안에 포함하여 패키징 가능하다.

<br>

#### 🍀 패키징

```
npm run build
```

정상적으로 패키징이 완료되면 `outputPath` 디렉터리에 바이너리(.exe) 파일이 생성된다.

![](/assets/tech/nodejs-binary-release-without-internet/image1.png)


<br>

#### 🍀 실행

`Release`된 바이너리를 실행하면 `CLI`가 팝업 되고 `express` 서버가 구동되었다는 내용이 출력된다. <br>
다음으로 브라우저에서 `GET` 메서드의 엔드 포인트에 접속해 보면 정상적으로 값이 리턴되는 것을 확인할 수 있다.

![](/assets/tech/nodejs-binary-release-without-internet/image2.png)
![](/assets/tech/nodejs-binary-release-without-internet/image3.png)

<br><br><br>

## ✨ pkg 폐쇄망 이슈(절망편)

앞서 말한 거처럼 내가 겪은 `Release` 환경은 폐쇄망이었기에 `node_modules`를 `Proxy`를 통해 다운받았다. <br>
하지만 빌드를 하게 되면 아래와 같이 일부 파일이 누락되었다고 에러가 발생한다. 🤬

```
> pkg@5.8.0
> Targets not specified. Assuming:
  node18-linux-x64, node18-macos-x64, node18-win-x64
> Fetching base Node.js binariees to PKG_CACHE_PATH
  fetched-v18.2.0-linux-x64            [] 0%> Not found in remote cache: 
  {"tag":"v3.4", "name":"node-v18.5.0-linux-x64"}
```

<br>

처음에는 파일 누락을 생각하지 못하고 `pkg` 모듈 동작은 무조건 인터넷을 필요로 하는 줄 알았다. 혹시 몰라서 와이어샤크로 아웃 바운드 패킷을 확인해 보니 `pkg`가 동작될 때 외부망으로 패킷이 발생하는 건 맞지만 [nodejs.org](nodejs.org)로 Request 하고 있었다. 

<br>

여기서 한 가지 가설(?)이 떠올랐다. [nodejs.org](nodejs.org)는 `Node.js`의 런타임이나 설정 파일을 다운로드하는 공식 사이트이다. `pkg` 모듈이 동작할 때 라이브러리 고유 URL이 아닌 [nodejs.org](nodejs.org)로 패킷을 전송하는 거라면 `pkg` 동작과 관련한 파일이 누락되어 이를 다운로드하기 위해 발생하는 패킷일 것이다. <br>
(이때 일부 라이브러리의 다운로드가 방화벽 때문에 누락되었다는 것을 인지함)

<br>

**그럼 해당 파일만 강제로 넣어주면 동작하지 않을까?** 

<br><br><br>

## ✨ 폐쇄망에서 pkg 사용 방법

어떤 파일을 어디에 넣어줘야 할지 막막했는데 생각보다 같은 문제를 겪은 사람들이 많았다. <br>
나는 해당 [포스팅](https://juejin.cn/post/7243276385896415291)을 참고하여 해결하였다. 

<br>

#### 🍀 파일 수동 다운로드

누군가 [Github](https://github.com/vercel/pkg-fetch/releases)에 `pkg` 개발 Assets을 업로드해 주었다. <br>
여기서 본인 환경에 맞는 `node-version-os-bit` 포맷 파일을 다운로드하자

![](/assets/tech/nodejs-binary-release-without-internet/image4.png)

<br>

#### 🍀 파일 수동 설정

다운로드한 파일을 어딘가에 넣어줘야 하는데 아까 발생했던 에러 로그가 힌트였다. 

```
{"tag":"v3.4", "name":"node-v18.5.0-linux-x64"}
```

`Windows OS` 기준 `C:\Users\{사용자명}\.pkg-cache\`에 `v3.4` 디렉토리를 생성하고 파일을 넣으라는 의미이다.<br> 
다운로드 파일을 넣을 때에는 파일명의 node를 fetched로 변경해 줘야 한다. 

<br>

#### 🍀 패키징

```
npm run build
```

다시 패키징 해보면 `outputPath` 디렉터리에 바이너리(.exe) 파일이 생성된다.

![](/assets/tech/nodejs-binary-release-without-internet/image1.png)

<br>

#### 🍀 실행

팝업 되는 `CLI`에서  `express` 서버 구동을 확인하고 브라우저에서 접속해 보면 정상 동작하는 것을 확인할 수 있다.

![](/assets/tech/nodejs-binary-release-without-internet/image2.png)
![](/assets/tech/nodejs-binary-release-without-internet/image3.png)

<br><br><br><br><br>

# 📌 References

<hr>

🔗 [Assets](https://github.com/vercel/pkg-fetch?tab=readme-ov-file)<br>
🔗 [Packaging](https://blog.outsider.ne.kr/1379)<br>
🔗 [Issue #1](https://stackoverflow.com/questions/67076767/nodejs-pkg-error-not-more-than-one-entry-file-directory-is-expected)<br>
🔗 [Issue #2](https://copycd.tistory.com/62)<br>
🔗 [Issue #3](https://www.reddit.com/r/node/comments/1bnfpdg/packaging_nodejs_script_to_exe/?rdt=53063)<br>
