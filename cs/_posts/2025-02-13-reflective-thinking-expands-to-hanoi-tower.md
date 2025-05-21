---
layout: post
title: 하노이의 탑으로 확장하는 재귀적 사고
description: >
  재귀를 활용한 하노이의 탑 풀이와 일반항 도출 방법에 대한 포스팅입니다.
sitemap: true
hide_last_modified: true
---

* toc
{:toc .large-only}

<br><br><br><br><br>

# 📌 Summary
<hr>

스터디에서 재귀 알고리즘 발표를 위해 `하노이의 탑` 문제를 학습하였는데, <br>
해당 문제는 **재귀**라는 개념을 이해할 수 있게 사고를 확장해 주는 좋은 내용인 거 같아서 기록한다.


<br><br><br><br><br>

# 📌 하노이의 탑

<hr>

하노이의 탑은 크기가 다른 원반들은 기둥 세 개에 옮기는 퍼즐 같은 놀이이다. <br>
단, 원반은 하나씩만 옮길 수 있으며 작은 원반 위에 큰 원반을 올릴 수 없다. 

![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image1.jpg)

<br><br><br>

## ✨ 하노이의 탑에서 도출할 항목

첫 번째 기둥에 있는 원반들을 세 번째 기둥으로 옮기는 문제를 가정하고 아래 항목들을 구해보자

1. 몇 번의 동작이 필요한지
2. 각 동작에서 n번 원반을 x기둥에서 y기둥으로 이동시켰는지

<br><br><br>

**🍀 원반이 1개인 경우** <br>

원반이 1개이면 1번의 동작이 필요하다. <br>

```
[ 1 ]　1번 원반이 1번 기둥에서 3번 기둥으로 이동
```

---|---|
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image2.jpg)|![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image3.jpg)

<br><br><br>

**🍀 원반이 2개인 경우** <br>

원반이 2개이면 3번의 동작이 필요하다. <br>

```
[ 1 ]　1번 원반이 1번 기둥에서 2번 기둥으로 이동
[ 2 ]　2번 원반이 1번 기둥에서 3번 기둥으로 이동
[ 3 ]　1번 원반이 2번 기둥에서 3번 기둥으로 이동
```

---|---|
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image4.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image5.jpg)
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image6.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image7.jpg)

<br><br><br>

**🍀 원반이 3개인 경우** <br>

원반이 3개이면 7번의 동작이 필요하다. <br>

```
[ 1 ]　1번 원반이 1번 기둥에서 3번 기둥으로 이동
[ 2 ]　2번 원반이 1번 기둥에서 2번 기둥으로 이동
[ 3 ]　1번 원반이 3번 기둥에서 2번 기둥으로 이동
[ 4 ]　3번 원반이 1번 기둥에서 3번 기둥으로 이동
[ 5 ]　1번 원반이 2번 기둥에서 1번 기둥으로 이동
[ 6 ]　2번 원반이 2번 기둥에서 3번 기둥으로 이동
[ 7 ]　1번 원반이 1번 기둥에서 3번 기둥으로 이동
```

---|---|
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_1.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_2.jpg)
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_3.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_4.jpg)
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_5.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_6.jpg)
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_7.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_8.jpg)

<br><br><br><br><br>

# 📌 하노이의 탑 풀이
<hr>

앞선 예제를 통해 어떤 방식으로 문제를 풀어야 할지 살펴보았다. <br>
이제 이것을 하나의 알고리즘으로 생각하고 어떤 코드로 풀이할 수 있을지 생각해 보자. 

<br>

하노이의 탑 알고리즘의 핵심은 **원반을 어떻게 이동시킬 것인가**이다. 

<br><br><br>

## ✨ 절차 지향적 접근(절망편)

위와 같이 원반의 개수가 정해져있는 경우라면, <br>
절차 지향적인 방법으로 조건문을 때려 박아 구현하는 것이 이론상 가능하다. 

<br>

하지만 그게 올바른 풀이 방법일까? <br>
모든 스텝을 나열하기 위해 수많은 조건을 고려해야 하며 코드는 원반의 개수에 비례하여 절망적으로 늘어난다. <br>

``` Python

if 조건:
    원반 이동
    if 조건:
        원반 이동
        if 조건:
            원반 이동
            if 조건:
                원반 이동
                if 조건:
                    원반 이동
                    if 조건:
                        원반 이동
                        if 조건:
                            원반 이동
                            if 조건:
                                원반 이동
                                if 조건:
                                    원반 이동
                                    if 조건:
                                        원반 이동

```

<br>

결론적으로 **문제의 본질에 접근하지 못한 풀이**이이다. 

<br><br><br>

## ✨ 재귀적 접근(희망편)

위 내용에서 중복되는 부분, 즉 자기 자신을 호출하여 재활용할 수 있는 부분을 찾는 걸로 접근해 보자.<br>
(원반을 옮기는 함수를 Move로 정의)

<br>

2개의 원반을 옮기는 경우 3번의 동작이 필요하다.<br>
**Move(2) = 3**이다.

![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/image4.jpg)

<br><br><br>

3개의 원반을 옮기는 경우 7번의 동작이 필요한데, 그 과정에서 Move(2)를 몇 번 활용할 수 있을지 살펴보자

1. 아래와 같이 시작 단계에서 1, 2번 원반을 2번으로 옮기는데 **Move(2)를 호출**한다. <br>
2. 다음으로 3번 원반을 1번 기둥에서 3번 기둥으로 옮기는데 **1번의 동작이 발생**한다. <br>
3. 마지막으로 2번 기둥에 위치한 1, 2번 원반을 3번으로 옮기는데 **Move(2)를 호출**한다. 

즉 **Move(3) = Move(2) + 1 + Move(2) = 7**과 같다. 


--|--|--|
![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_1.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_5.jpg) | ![](/assets/cs/reflective-thinking-expands-to-hanoi-tower/c3_8.jpg)

<br><br><br>

이를 통해 **Move(N) = Move(N - 1) * 2 + 1**이라는 수식을 도출했다. 

Move(1) = Move(0) * 2 + 1 = 1<br>
Move(2) = Move(1) * 2 + 1 = 3<br>
Move(3) = Move(2) * 2 + 1 = 7<br>
Move(4) = Move(3) * 2 + 1 = 15<br>
... 

<br><br><br><br><br>

# 📌 하노이의 탑 재귀로 구현하기
<hr>

하노이의 탑 문제에서 재귀적 관점을 찾아냈다면 이제 코드로 구현해 보자

<br>

먼저 원반을 이동하는 함수의 파라미터로 3개를 정의해야 한다. <br>
n번 원반이 x번 기둥에서 y번 기둥으로 이동하는 것을 파악하기 위해서이다.
``` python
def move(n, x, y)
```

<br>

다음은 재귀를 통해 원반들을 이동시켜야 하는데, 호출점을 파악해 보자. <br>
우선 맨 위에 있는 원반부터 차례로 이동시키기 위해 원반 번호는 n - 1로 재귀를 호출한다. <br>
x는 초기에 원반들이 위치한 곳이 1번 기둥이기 때문에 1을 전달하며, <br> 
y에는 세 기둥 번호의 합인 6에 x, y 값을 뺀 값을 전달한다. <br>

예를 들어 출발 1, 도착이 3이면 원반은 6 - 1 - 3 = 2에 쌓인다.<br> 
출발이 1, 도착이 2이면 6 - 1 - 2 = 3에 쌓이게 된다. 
위에 있는 원반들부터 순차적으로 다른 기둥에 쌓아두기 위한 재귀 호출점이다.

``` python
def move(n, x, y): 
    if n > 1:
        move(n - 1, x, 6 - x - y)

    print(f"{n}번 원반이 {x}번 기둥에서 {y}번 기둥으로 이동")
```

<br>

하지만 위 코드를 실행하면 아래와 같이 출력된다. 

```
1번 원반이 1번 기둥에서 3번 기둥으로 이동
2번 원반이 1번 기둥에서 2번 기둥으로 이동
3번 원반이 1번 기둥에서 3번 기둥으로 이동
```

<br>

1번 원반 위에 더 큰 3번 원반 위에 쌓였고, 이동도 마무리되지 않았다.  <br> 
출발 기둥과 도착 기둥이 아닌, 다른 보조 기둥에 쌓아뒀던 원반들을 다시 이동시키는 재귀 호출점을 추가로 설정하자. <br>

x와 y의 파라미터를 바꾸어야 한다. <br>
보조 기둥이었던 6 - x - y는 출발 기둥이 되고 최종 도착 기둥은 y가 되어야 하기 때문이다. 
``` python
def move(n, x, y): 
    if n > 1:
        move(n - 1, x, 6 - x - y)

    print(f"{n}번 원반이 {x}번 기둥에서 {y}번 기둥으로 이동")

    if n > 1:
        move(n - 1, 6 - x - y, y)
```

<br>

최종 코드는 아래와 같다.

``` python
"""
하노이의 탑 재귀 함수

:param n: 원반
:param x: 출발 기둥
:param y: 도착 기둥
"""
def move(n, x, y):
    if n > 1:
        move(n - 1, x, 6 - x - y)

    print(f"{n}번 원반이 {x}번 기둥에서 {y}번 기둥으로 이동")

    if n > 1:
        move(n - 1, 6 - x - y, y)


if __name__ == "__main__":
    N = 3  
    move(N, 1, 3) 
```

```
출력: 
1번 원반이 1번 기둥에서 3번 기둥으로 이동
2번 원반이 1번 기둥에서 2번 기둥으로 이동
1번 원반이 3번 기둥에서 2번 기둥으로 이동
3번 원반이 1번 기둥에서 3번 기둥으로 이동
1번 원반이 2번 기둥에서 1번 기둥으로 이동
2번 원반이 2번 기둥에서 3번 기둥으로 이동
1번 원반이 1번 기둥에서 3번 기둥으로 이동
```

P.S. 이해가 어렵다면 손 코딩으로 재귀 호출 지점과 파라미터를 그려보는 것을 추천한다. <br>

<br><br><br><br><br>

# 📌 하노이의 탑 일반항 구하기(수학)
<hr>

## ✨ 일반항을 왜 구해야 할까?

하노이의 탑 일반항을 구해기 앞서 이걸 왜 구해야 하는지 알아보자. 

앞선 내용으로 **Move(N) = Move(N - 1) * 2 + 1**라는 수식과 재귀적 관점을 발견했고, 이를 코드로 구현해 보았다. 

<br>

하지만 해당 수식으로 Move(100)을 구할 경우, Move(99)를 호출해야 하고, Move(99)는 또다시 Move(98)을 호출... Move(1) 까지 결국 모두 호출해야 한다. 
<br>
일반항을 구할 수 없거나 DP 등의 알고리즘 적용이 불가한 경우에는 어쩔 수 없지만, 그게 아니라면 복잡도 측면에서 좋지 않은 해결 방법이다.

<br>

**일반항만 구할 수 있다면 재귀로 자기 자신을 호출하지 않아도 된다.**

<br><br><br>

## ✨ 일반항 구하기

Move(N) = 2 * Move(N-1) + 1<br>
Move(N-1) = 2 * Move(N-2) + 1
∴ Move(N) = 2^2 * Move(N-2) + 2 + 1 

<br>

Move(N-2) = 2 * Move(N-3) + 1 <br>
∴ Move(N) = 2^3 * Move(N-3) + 2^2 + 2^1 + 1

<br>

∴ Move(N) = 2^(N-1) + 2^(N-2) + ... + 2^2 + 2^1 + 2^0

<br>

이는 아래와 같은 등비수열의 합 공식으로 정리할 수 있다.

Move(N) = 1 + 2 + 2^2 + ... + 2^(N-2) + 2^(N-1)

<br>

등비수열의 합 공식을 적용하면 일반항을 도출할 수 있다.  <br>
(첫째항이 a, 공비 r, 항의 개수 n 일 때 합 S = a * (r^(n - 1)) / (r - 1))

Move(N) = 2^N - 1

<br><br><br>

## ✨ 일반항 검증

Move(1) = 2^1 - 1 = 1<br>
Move(2) = 2^2 - 1 = 3<br>
Move(3) = 2^3 - 1 = 7<br>
Move(100) = 2^100 - 1 = 1267650600228229401496703205375

<br><br><br><br><br>

# 📌 References 
<hr>

🔗 [하노이의 탑 설명](https://shoark7.github.io/programming/algorithm/tower-of-hanoi)<br>
🔗 [하노이의 탑 구현1](https://www.geeksforgeeks.org/c-program-for-tower-of-hanoi/)<br>
🔗 [하노이의 탑 구현2](https://medium.com/swlh/coding-problems-why-do-i-like-them-solving-tower-of-hanoi-puzzle-d1bd71f85982)<br>