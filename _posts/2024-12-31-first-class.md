---
title: First-Class function/object란?
description: What is First-Class
author: yw1ee
categories: [CS]
tags: [first-class, computer science, sicp]
---

## First-Class citizen

SICP 1.3.4장을 보면 first-class procedure라는 말이 나온다. first-class라는 말을 여기저기서 들어보기도 했고, 얼추 어떤 뜻인지는 알고 있지만 이 말을 공부해본적은 없는 것 같아 정리해보려 한다.

First-Class function의 뜻을 알기 위해서는 first-class citizen을 먼저 알아야 하는데, CS의 programing language design에서 first-class citizen이란 인자로 넘기기, 변수에 할당하기, 함수로부터 리턴되기 등의 동작들이 기본적으로 지원되는 객체를 의미한다. 내가 이해한 바에 의하면 언어에 기본적으로 정의된 primitive type들을 뜻한다. 예를 들면 boolean이나 int 등등.

## First-Class function

함수를 First-Class citizen으로 다룰 수 있는 언어에서의 함수를 First-Class function 라고 부른다. 굉장히 많은 언어들의 함수가 First-Class function인데 친숙한 Python 또한 마찬가지이다.

```python
def return_f():
    def func(a ,b):
        return a + b
    return func

def run_f(f, arg1, arg2):
    return f(arg1, arg2)

"""함수를 변수에 할당하기"""
f = return_f
print(f'"f" is function: {f}')
# "f" is function: <function return_f at 0x1001614e0>

"""함수를 리턴하기"""
func = f()
print(f'returned value "func" is function: {func}')
# returned value "func" is function: <function return_f.<locals>.func at 0x100228540>

"""함수를 인자로 넘기기"""
ret = run_f(func, 1, 2)
print(f'1 + 2 = {ret}')
# 1 + 2 = 3
```

### C언어의 함수는 First-Class function일까?

정답은 아니다. 몇몇 저자들은 런타임에 새로운 함수를 생성할 수 있어야 First-Class라고 부를 수 있다고 한다. 이 정의에 따르면 C에서의 함수는 First-Class가 아니다. 다만 function pointer를 통해 위에 정의한 동작들을 비슷하게 수행할 수 있다는 점에서 Second-Class function이라고 종종 불린다고 한다.

또 [stack overflow의 또 다른 답변](https://stackoverflow.com/questions/10777333/functions-are-first-class-values-what-does-this-exactly-mean/10782920#10782920)에 의하면 First-Class citizen은 아래 세가지 조건을 충족해야 하는데, C언어의 함수는 (논란의 여지가 있지만) 1, 3번 조건을 fucntion pointer를 통해 만족한다 하더라도 2번 조건을 만족할 수 없기 때문에 (local variable로 함수를 생성할 수 없다) First-Class fuction이 아니다.
1. 아무 제한 없이 함수의 인자나 리턴 값으로 사용되거나 containter에 담길 수 있어야 함
2. 아무 제한 없이 어디서든 생성될 수 있어야 함
3. 일반적은 값들 처럼 타입을 지정할 수 있어야 하고, 다른 타입들과 함께 구성될 수 있어야 함

## 또 다른 First-Class objects

다른 First-Class object들 중 흥미로운 두 가지를 조금 살펴봤다.
- First-Class class
  - class 타입의 객체가 아닌, class 자체를 First-Class object로 다룬다
- First-Class control
  - "코드가 특정 지점까지 실행된 상태" 를 하나의 객체로 나타내고, 이를 First-Class citizen으로 다룬다. 예를 들어 이를 활용하면 multi-thread 환경에서의 coroutine을 구현할 수 있다. 다른 언어들에서도 coroutine처럼 비슷한 기능들이 지원되지만 다른 라이브러리의 구현에 기대어 동자하는 것이지, First-Class citizen으로 취급하지는 않는다는 점을 주의해야 한다.

## 참고
- [https://stackoverflow.com/questions/10777333/functions-are-first-class-values-what-does-this-exactly-mean/10782920#10782920](https://stackoverflow.com/questions/10777333/functions-are-first-class-values-what-does-this-exactly-mean/10782920#10782920)
- [https://en.wikipedia.org/wiki/First-class_citizen](https://en.wikipedia.org/wiki/First-class_citizen)
- [https://en.wikipedia.org/wiki/First-class_function](https://en.wikipedia.org/wiki/First-class_function)