---
layout: post
title: Python 3.15 버전에서 업데이트된 샘플링 프로파일러 사용해 보기
description: >
  Python 3.15 버전부터 업데이트된 샘플링 프로파일러의 기능과 동작 방식에 대한 포스팅입니다.
sitemap: true
hide_last_modified: true
---

* toc
{:toc .large-only}

<br><br><br><br><br>

# 📌 Overview

<hr>

`Python 3.15` 부터 추가되는 샘플링 프로파일링을 어떻게 활용하는지, 
`cProfile` 등 기존 결정론적 프로파일링 도구들과 비교했을 때 구체적으로 어떤 점들이 개선되었는지를 기록한다. 

<br><br><br><br><br>

# 📌 프로파일링이란?

<hr>

<details>
    <summary>👉 프로파일링에 대한 주관적 관점</summary>

<br>

최근들어 서비스 개발을 할 때에는 기능 구현보다 설계와 성능에 투자하는 시간이 늘어나고 있는 거 같다. 
단순 기능 구현은 AI가 작성해주는 다양한 코드 중에서 입맛에 맞게 선택하고 적용하면 되기 때문에, 
그만큼 확보하는 시간을 설계 방식이나 성능 최적화를 고민하는데에 할애할 수 있다. 
이렇듯 서비스의 완성도와 직결되는 이점이 AI의 순기능이 아닐까 싶다.

<br><br>

그렇다면 설계에 문제는 없는지, 성능에 대한 최적화는 필요하지 않은지를 판단할 수 있는 방법은 무엇일까? 
가장 정확한 방법은 `프로파일링(Profiling)` 기법을 사용하는 것이다.

<br><br>

</details>

<br>

`프로파일링(Profiling)`은 `프로파일러(Profiler)`라는 도구를 사용하여 소스 코드나 바이너리 파일을 계측 분석하는 기법이다. 
프로그램의 시간/메모리 복잡도나 특정 함수의 호출 주기 및 빈도를 측정할 수 있다. 
이를 통해 프로그램의 실행 시간, CPU 사용량, 메모리 소비량 등의 세부 사항을 파악하여 병목 구간을 발굴하고 로직을 최적화한다. 

- <b>성능 병목 현상 파악</b>: 프로그램에서 CPU 시간, 메모리, I/O 등 특정 리소스를 과도하게 사용하는 부분을 찾아낸다. 어떤 함수가 실행 시간의 대부분을 차지하는지, 어떤 코드가 메모리 누수를 발생시키는지 등을 분석할 수 있다.
- <b>리소스 사용량 최적화</b>: 프로그램이 어떤 리소스를 얼마나 사용하는지 측정하여 불필요한 리소스 사용을 줄이고 효율성을 높이는 방법을 모색한다.
- <b>코드 개선점 발굴</b>: 특정 코드 블록이나 함수의 실행 빈도, 호출 시간 등을 분석하여 성능 개선이 필요한 부분을 발굴한다.

<br><br><br><br><br>

# 📌 cProfile을 이용한 프로파일링

`Python 3.15` 이전까지는 내장 모듈인 `cProfile`을 이용하여 프로파일링 하는 것이 일반적이다. 
`cProfile`은 결정론적 프로파일링(Deterministic Profiling) 방식으로 작동한다.
프로그램이 실행되는 동안 모든 함수 호출, 함수 반환, 예외 발생 등 주요 이벤트를 하나도 놓치지 않고 추적하여 상세한 통계를 기록하는데, 
함수 자체의 실행 시간인 `tottime`와 함수가 호출된 시점부터 종료될 때까지 걸린 전체 시간인 `cumtime` 두 가지를 핵심 항목으로 사용한다. 


<br><br><br>

## ✨ cProfile 사용해 보기

우선 프로파일링 대상 코드를 살펴보자. 
`HTTP GET`으로 아이템 ID를 입력받아 해당하는 아이템 정보를 반환하는 간단한 API이다. 

``` Python 
# main.py
@app.get("/items/{item_id}", response_model=ResponseModel)
async def get_info(item_id: str) -> Dict[str, str]:
    
    # 아이템 정보 조회를 위한 API 호출
    result = await get_data_from_api(item_id)

    if result is None:
        raise HTTPException(status_code=400, detail="조회 실패")
    
    return {"result": result, "message": "성공"}
```

<br><br>

위 코드에서 `get_data_from_api` 함수 호출이 병목 구간으로 의심될 경우, 
해당 함수의 순수 실행 시간을 알아내고자 프로파일링을 해볼 수 있다. 
함수 내부에서 어떤 또 다른 함수를 호출하는지, 그 함수의 처리 시간은 얼마나 소요되었는가로 정확한 병목 구간을 파악할 수 있는 것이다. 

<br><br>

이제 `cProfile` 모듈로 `get_data_from_api` 함수를 프로파일링 하기 위한 코드를 작성해 보자. 

> `cProfile` 모듈은 비동기 함수의 프로파일링을 지원하지 않는다. 
따라서 프로파일링 대상인 비동기 함수 `get_data_from_api`를 호출하기 위한 래퍼(Wrapper) 함수로 `run_async_profiling`를 생성해 주었다.

``` Python
# profiling.py
from src.database.db import get_data_from_api
from cProfile import Profile
from pstats import Stats
import asyncio

profiler = Profile()

def run_async_profiling():
    asyncio.run(get_data_from_api("test"))

profiler.runcall(run_async_profiling)


# 프로파일링 결과 통계
# cumtime으로 정렬
stats = Stats(profiler)
stats.strip_dirs()
stats.sort_stats(cumtime)
stats.print_stats()
```

<br><br>

`python profiling.py`으로 실행해 보면 총 몇 개의 함수가 몇 초 동안 호출되었는지를 파악할 수 있다.

```
    551 function calls (549 primitive calls) in 0.001 seconds

    Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.001    0.001 profiling.py:8(run_async_profiling)
        1    0.000    0.000    0.001    0.001 runners.py:160(run)
        1    0.000    0.000    0.000    0.000 runners.py:58(__enter__)
        2    0.000    0.000    0.000    0.000 runners.py:131(_lazy_init)
        1    0.000    0.000    0.000    0.000 events.py:808(new_event_loop)
        1    0.000    0.000    0.000    0.000 events.py:693(new_event_loop)
        1    0.000    0.000    0.000    0.000 windows_events.py:312(__init__)
        1    0.000    0.000    0.000    0.000 proactor_events.py:631(__init__)
        3    0.000    0.000    0.000    0.000 base_events.py:618(run_until_complete)
        3    0.000    0.000    0.000    0.000 windows_events.py:317(run_forever)
        1    0.000    0.000    0.000    0.000 proactor_events.py:782(_make_self_pipe)
        1    0.000    0.000    0.000    0.000 socket.py:616(socketpair)
        1    0.000    0.000    0.000    0.000 runners.py:86(run)
    ... 
```

<br>

`stats.sort_stats()`로 정렬 기준도 설정할 수 있다.

- `calls(ncalls)`: 함수 호출 횟수
- `tottime(time)`: 함수 자체의 실행 시간 (하위 함수 제외)
- `cumtime`: 함수와 그 하위 함수를 포함한 누적 실행 시간
- `pcalls`: 기본 호출 횟수 (재귀 호출 포함)
- `filename`: 파일 이름
- `line`: 라인 번호
- `name`: 함수 이름

<br><br><br>

## ✨ Delay가 발생하는 함수 프로파일링 해보기

이번에는 함수의 부하를 가정하고 함수를 `return` 하기 전 지연을 설정해 보았다.

``` Python
# db.py
import asyncio
import random

async def get_data_from_api(item_id: str) -> dict:
    # 0.5 ~ 0.9초 사이의 지연 발생
    await asyncio.sleep(round(random.uniform(0.5, 0.9), 4))
    
    """
    DB 쿼리 생략
    """
    return {"item_id": item_id, "data": "Test Data"}
```

<br><br>

`profiling.py`를 다시 실행해 보면 `profiling.py:8(run_async_profiling)`의 처리 시간이 증가한 것을 확인할 수 있다.

```
    617 function calls (615 primitive calls) in 0.850 seconds

    Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.850    0.850 profiling.py:8(run_async_profiling)
        1    0.000    0.000    0.850    0.850 runners.py:160(run)
        3    0.000    0.000    0.849    0.283 base_events.py:618(run_until_complete)
        3    0.000    0.000    0.848    0.283 windows_events.py:317(run_forever)
        3    0.000    0.000    0.848    0.283 base_events.py:594(run_forever)
        8    0.000    0.000    0.848    0.106 base_events.py:1859(_run_once)
        1    0.000    0.000    0.847    0.847 runners.py:86(run)
        9    0.000    0.000    0.847    0.094 windows_events.py:812(_poll)
        8    0.000    0.000    0.847    0.106 windows_events.py:442(select)
       12    0.846    0.071    0.846    0.071 {built-in method _overlapped.GetQueuedCompletionStatus}
        1    0.000    0.000    0.002    0.002 runners.py:62(__exit__)
        1    0.000    0.000    0.002    0.002 runners.py:65(close)
       13    0.000    0.000    0.001    0.000 events.py:82(_run)
    ... 
```

<br><br><br>

## ✨ cProfile의 단점

하지만 `cProfile`과 같은 결정론적 프로파일링 방식은 근본적으로 단점이 존재한다. 프로그램의 실행 과정에 후크(hook)를 걸어 각 함수가 호출되거나 반환될 때마다 데이터를 기록하기 때문에 실행 내용 모두를 정확하게 파악할 수 있다는 장점이 있지만 <b>오버헤드가 높고 실제 프로덕션 서비스에 적용하기 어렵다.</b>

<br>

모든 함수 호출을 추적하는 과정 자체가 많은 리소스를 소모하기 때문에 호출 빈도가 높은 함수에서는 성능 저하가 발생할 수 있다. 
따라서 프로덕션 환경에서만 발생하는 성능 이슈를 파악하기 위해 적용하는 것은 적합하지 않다.

<br><br><br><br><br>

# 📌 Python 3.15의 샘플링 프로파일링

(3.15 출시 시 이어서 작성 예정)

`Python 3.15`부터 `profile.sample`이 새롭게 추가되었다. 
[공식 문서](https://docs.python.org/3.15/whatsnew/3.15.html)에서는 High frequency statistical sampling profiler 즉 <b>고주파 통계 샘플링 프로파일러</b>로 소개되었다. 

<br>

기존에 사용되던 결정론적 프로파일러인 `profile`, `cProfile`과 달리 `profile.sample`는 실행되는 프로세스에서 주기적으로 모든 함수 호출의 스택 추적을 캡처한다. 
이러한 접근 방식은 오버헤드가 거의 없는 동시에 최대 1,000,000Hz의 샘플링 속도로 현재까지 `Python`에서 가장 빠른 샘플링 프로파일러 기법이라고 한다.

