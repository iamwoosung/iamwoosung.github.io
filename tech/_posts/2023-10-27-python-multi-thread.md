---
layout: post
title: 파이썬 멀티 스레드와 병렬 처리
description: >
  파이썬의 멀티 스레드 동작 방식과 병렬 처리 방법에 대한 포스팅입니다.
sitemap: true
hide_last_modified: true
tags: [Python, CS]
---

* toc
{:toc .large-only}

<br><br><br><br><br>

# 📌 개요

<hr>

학부 고급 프로그래밍 수업에서 `Python`과 관련한 주제로 발표를 하게 되었고, `Multi-Thread`의 동작 방식과 `병렬처리` 방법에 대한 학습 내용을 기록한다.

<br><br><br><br><br>

# 📌 프로세스(Process)와 스레드(Thread)

<hr>

핵심 내용 설명에 앞서 `프로세스`와 `스레드`를 이해해야 한다.

<br>

`프로세스`란 실행 중인 프로그램이다. 
- OS로부터 CPU, Memory 등의 자원을 할당받고 관리된다.
- 고유한 Process ID를 갖는다.
(작업 관리자의 세부 정보 탭에서 프로세스 및 ID 확인이 가능하다.)
![](/assets/tech/python-multi-thread-and-parallel-processing/image1.png)

<br>

`스레드`란 실행 흐름의 단위이다.
- `프로세스`에서 할당받은 자원을 이용한다.
- `프로세스` 내에 존재하는 단위이며 하나면 `싱글 스레드`, 다수면 `멀티 스레드`이다.

<br><br><br><br><br>

# 📌 멀티 스레드란 무엇인가

<hr>

예시로, 3개 파일을 다운로드 하려고 한다.

<br>

`싱글 스레드`의 경우, 한 번에 하나의 파일만을 **순차적으로** 다운로드해야 한다.
`멀티 스레드`의 경우, 여러 파일을 **동시에** 각각의 스레드에서 다운로드할 수 있다. 

<br>

이를 각각 **`순차 실행(Sequential)`**과 **`병렬 실행(Concurrent)`**이라고 한다.

![](/assets/tech/python-multi-thread-and-parallel-processing/image2.jpg)



위와 같은 동작이 일반적으로 생각하는 `멀티 스레드`이다. 
하지만 결론부터 얘기하면 **`Python`의 `스레드`에서는 위와 같은 동작이 불가하다.**

<br><br><br><br><br>

# 📌 언어별 스레드 비교와 파이썬 GIL

<hr>

`Python`에서 `멀티 스레드`는 가능하지만 `병렬처리`는 불가하다.
어떤 의미인지 아래 예제를 통해 확인해 보자.

<br>

0~100000000까지 합 출력 코드이다.
`싱글 스레드`는 0~100000000까지 하나의 `스레드`에서 계산을 수행한다. 
`멀티 스레드`는 50000000씩 두 개의 `스레드`에서 계산을 수행한다. 

<br>

P.S. 정확한 수치를 원한다면 CPU 성능을 낮추어 자원에 자체적인 부하와 제약을 주고 시험할 수 있다.
제어판 → 하드웨어 및 소리 → 전원 옵션 → 전원 관리 옵션 설정 편집 → 고급 전원 관리 옵션 설정 변경
![](/assets/tech/python-multi-thread-and-parallel-processing/image3.png)

<br>

## ✨ Python 싱글 스레드

실행 시간: 4,500 ms

``` python
import time
from threading import Thread

def work(id, start, end, result):
    total = 0
    for i in range(start, end):
        total += i
    result.append(total)
    return

if __name__ == "__main__":

    start = time.time()

    result = list()
    th1 = Thread(target=work, args=(1, 0, 100000000, result))
    
    th1.start()
    th1.join()

print(f"Result: { sum(result) }")
print(f"Time: { time.time() - start }")
```

<br>

## ✨ Python 멀티 스레드

실행 시간: 4,500 ms

``` python
import time
from threading import Thread

def work(id, start, end, result):
    total = 0
    for i in range(start, end):
        print(i)
        total += i
    result.append(total)
    return

if __name__ == "__main__":

    start = time.time()


    result = list()
    th1 = Thread(target=work, args=(1, 0, 50000000, result))
    th2 = Thread(target=work, args=(2, 50000000, 100000000, result))
    
    th1.start()
    th2.start()
    th1.join()
    th2.join()

print(f"Result: { sum(result) }")
print(f"Time: { time.time() - start }")

print(len(result))
```

<br>

## ✨ Java 싱글 스레드

실행 시간: 4,600 ms

``` java
import java.util.ArrayList;

public class Main {
	
	public static void main(String[] args) throws Exception {
		new Main();
	}
	
	public Main() throws Exception {
		long startTime = System.currentTimeMillis();

		
		TestThread th = new TestThread(0, 100000000);
		th.start();
		th.join();
		
		long time = System.currentTimeMillis() - startTime;
		System.out.println((double) time / 1000);
	}
	
	class TestThread extends Thread {

		private ArrayList<Integer> list = new ArrayList<Integer>();
		private int start;
		private int end;
		

		public TestThread(int start, int end) {
			this.start = start;
			this.end = end;
		}

		@Override
		public void run() {
			long sum = 0;
			for (int i = start; i < end; i++) {
				list.add(i);
			}
			
			for(int d : list) sum += d;
			System.out.println("sum: " + sum);
			
		}
	}

}
```

<br>

## ✨ Java 멀티 스레드

실행 시간: 3,500 ms

``` java
import java.util.ArrayList;

public class Main {
	
	public static void main(String[] args) throws Exception {
		new Main();
	}
	
	public Main() throws Exception {
		long startTime = System.currentTimeMillis();
		
		TestThread th1 = new TestThread(0, 50000000);
		TestThread th2 = new TestThread(50000000, 100000000);
		th1.start();
		th2.start();
		th1.join();
		th2.join();
		
//		while(th.isAlive()) {}
		
		long time = System.currentTimeMillis() - startTime;
		System.out.println((double) time / 1000);
	}

	
	
	
	class TestThread extends Thread {

		private ArrayList<Integer> list = new ArrayList<Integer>();
		private int start;
		private int end;
		

		public TestThread(int start, int end) {
			this.start = start;
			this.end = end;
		}

		@Override
		public void run() {
			long sum = 0;
			for (int i = start; i < end; i++) {
				list.add(i);
			}
			
			for(int d : list) sum += d;
			System.out.println("sum: " + sum);
			
		}
	}

}
```

<br>

## ✨ 내용 정리

`Java`의 경우 `멀티 스레드`가 `싱글 스레드` 보다 확연히 빠르다. 
`Python`의 경우 `멀티 스레드`와 `싱글 스레드`의 수행 속도가 거의 동일하다.

<br>

언어별로 실행 시간에 차이가 발생하는 것은 당연하다. 
하지만, `Python`은 멀티 스레드와 싱글 스레드의 속도가 동일하다는 것을 통해 `병렬 실행`이 아닌 `순차 실행`되고 있다는 것을 증명할 수 있다.

<br>

해당 방식으로 동작하는 이유는 무엇일까?

> 위키에 따르면 `Python`에는 `GIL(Global Interpreter Lock)`이라는 녀석이 자원 관리를 목적으로 인터프리터가 반드시 하나의 `스레드`만을 수행하도록 제한해 놓았다고 한다. 

(GIL은 기회가 된다면 별도 게시물에서 다룰 예정)

![](/assets/tech/python-multi-thread-and-parallel-processing/image4.png)


<br><br><br><br><br>

# 📌 파이썬 스레드의 활용

<hr>

앞선 내용으로 `Python`의 `스레드`를 이해했다. 
그렇다면 `병렬처리`가 불가한 `스레드`를 어떻게 활용할 수 있을까?

<br>

**I/O 작업**이 많을 경우이다. 

<br>

`Python`은 I/O 작업이 발생 시, 
`스레드`의 실행을 중지하고 해당 작업 종료까지 대기 상태로 빠진다. 

<br>

즉, `멀티 스레드`에서 사용 중인 `스레드`가 대기 상태로 빠지면 다른 `스레드`의 작업을 수행하게 되는 것이다. 

<img src="/assets/tech/python-multi-thread-and-parallel-processing/image5.jpg" width="30%">


디스크가 파일을 읽고 쓰거나 네트워크 카드(NIC)가 송/수신할 때와 같이, 
CPU가 아닌 하드웨어에서 발생되는 연산을 효율적으로 처리할 수 있다. 

Web, REST API 서버, 대용량 로그 생성 등의 개발 시 활용 가능하다.

<br><br><br><br><br>

# 📌 파이썬 병렬 처리

<hr>

`Python` `스레드`로는 `병렬처리`가 불가하다는 사실을 알게 되었다. 
그렇다면 `Python`에서 `병렬처리` 구현이 필요한다면 어떻게 해야 할까

<br>

**`멀티 프로세스`를 사용하면 된다.**
multiprocessing 모듈을 사용하여 하나의 프로그램에 다수 프로세스를 생성/실행할 수 있다. 

<br>

아래 예제 코드를 실행해 보면 실행 시간이 단축되는 것을 알 수 있고, 
`멀티 프로세스`의 경우 실행 동안 작업 관리자에서 PID가 다중으로 생성되는 것을 확인할 수 있다.

<br>

## ✨ Python 싱글 프로세스

실행 시간: 4,500 ms

``` python
import time
from multiprocessing import Process, Queue

def work(id, start, end, result):
    total = 0
    for i in range(start, end):
        total += i
    result.put(total)
    return

if __name__ == "__main__":
    start = time.time()
    
    START, END = 0, 100000000
    result = Queue()
    th1 = Process(target=work, args=(1, START, END, result))
    
    th1.start()
    
    th1.join()
    

    result.put('STOP')
    total = 0
    while True:
        tmp = result.get()
        if tmp == 'STOP':
            break
        else:
            total += tmp

    print(f"Result: {total}")
    print(f"Time: { time.time() - start }")
```

<br>

## ✨ Python 멀티 프로세스

실행 시간: 3,800 ms

``` python
import time
from multiprocessing import Process, Queue

def work(id, start, end, result):
    total = 0
    for i in range(start, end):
        total += i
    result.put(total)
    return

if __name__ == "__main__":
    start = time.time()
    
    START, END = 0, 100000000
    result = Queue()
    th1 = Process(target=work, args=(1, START, END//2, result))
    th2 = Process(target=work, args=(2, END//2, END, result))
    
    th1.start()
    th2.start()
    th1.join()
    th2.join()

    result.put('STOP')
    total = 0
    while True:
        tmp = result.get()
        if tmp == 'STOP':
            break
        else:
            total += tmp

    print(f"Result: {total}")
    print(f"Time: { time.time() - start }")
```

<br><br><br><br><br>

# 📌 결론

<hr>

`Python`에서 `병렬처리`는 `멀티 프로세스`로 구현이 가능하다. 
`멀티 스레드`는 결국 하나의 스레드를 수행하므로 `병렬처리`가 되지 않는다.

<br><br><br><br><br>

# 📌 내용 추가

<hr>

(2024-07-12) 내용 추가: GIL 사용을 선택 사항으로 변경 예정

![](/assets/tech/python-multi-thread-and-parallel-processing/image6.png)

<br><br><br><br><br>

# 📌 References

<hr>

🔗 [멀티 스레드](https://xangmin.tistory.com/175)<br>
🔗 [병렬 처리](https://docs.python.org/ko/3.8/library/threading.html)<br>
🔗 [GIL](https://it-eldorado.tistory.com/160)<br>
🔗 [GIL](https://yscho03.tistory.com/295)<br>