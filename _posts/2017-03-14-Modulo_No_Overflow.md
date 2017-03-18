---
layout: post
title:  "오버플로우 없는 나머지 연산 구현하기"
date:   2017-03-14 21:24:27 +0900
categories: Programming
---

## 간단한 정수론 복습

이번 포스트에서는 정수의 나머지와 관련되어 발생할 수 있는 오버플로우를 방지하는 방법에 대해 정리하고자 합니다. 사실 프로그래밍을 하다 보면 생각 외로 정수의 나머지를 구하는 연산을 할 때가 많습니다. 그래서 정수의 나머지와 관련된 몇 가지 성질을 복습하는 것도 의미가 있을 것 같습니다.

먼저 대부분의 언어에서 정수의 나머지를 구하는 연산자는 ```%```입니다. 그런데 수학에서는 ```%``` 기호 대신 $$\textrm{mod}$$ 라고 표기합니다. 

* a를 n으로 나눈 나머지는 $$a \: \textrm{mod} \: n$$
* a와 b를 n으로 나눈 나머지가 같으면 $$a \equiv b (\textrm{mod} \: n)$$
* 더하거나 빼도 관계는 유지됩니다 $$a \equiv b (\textrm{mod} \: n) \iff a + c \equiv b + c (\textrm{mod} \: n)$$
* 만일 $$k$$와 $$n$$가 서로소라면 $$a \equiv b (\textrm{mod} \: n) \iff ak \equiv bk (\textrm{mod} \: n)$$

## 더하기에서 오버플로우 방지하기
두 정수 $$a$$와 $$b$$를 더한 수를 $$n$$으로 나눈 나머지를 구하는 코드는 다음과 같습니다.
{% highlight C %}
unsigned long long c = (a + b) % n
{% endhighlight %}

그런데 $$a$$와 $$b$$가 아주 큰 숫자여도 이게 성립할까요?
{% highlight C %}
unsigned long long a = ~0; // 2^64 - 1
unsigned long long b = ~0; // 2^64 - 1
unsigned long long n = ~0; // 2^64 - 1
unsigned long long c = (a + b) % n;
// c == 18446744073709551614
{% endhighlight %}

코드를 돌려보면 $$c = 18446744073709551614$$라고 나옵니다. 이건 말이 안 됩니다. 왜냐하면 $$a$$,$$b$$,$$n$$을 모두 똑같은 숫자로 줬기 때문에 $$c = 0$$이 되어야 하거든요. 이 문제가 발생한 건 ```a+b```에서 오버플로우가 발생했기 때문입니다.

Swift에서는 오버플로우가 발생하면 런타임에 프로그램이 뻗어버리지만, C에서는 그냥 오버플로우가 난 채로 잘못된 값이 들어가버립니다. 그렇기 때문에 오버플로우는 반드시 교정되어야 합니다. 이 문제는 다음의 코드로 교정할 수 있습니다.
{% highlight C %}
//http://stackoverflow.com/a/10077138
unsigned long long addmod(unsigned long long a, unsigned long long b, unsigned long long n)
{
    unsigned long long amod = a % n;
    unsigned long long bmod = b % n;
    
    if (amod == 0) {
        return bmod;
    }
    if (bmod == 0) {
        return amod;
    }
    if (amod + bmod <= amod) {
        return (amod - (n - bmod)) % n;
    }
    
    return (amod + bmod) % n;
}
{% endhighlight %}

이 코드가 제대로 동작하는 이유는 다음과 같습니다. 첫째, 오버플로우의 탐지입니다. ```amod + bmod <= amod```가 오버플로우를 탐지하는 이유는, 오버플로우가 발생하면 원래의 숫자보다 작아지기 때문입니다. 4비트짜리 정수 ```a = 1011```, ```b = 1110```을 예로 들면, ```a+b = 10001```이 되는데 맨 왼쪽의 비트는 버려집니다. 그래서 ```a+b = 0001```이 됩니다. 둘째, $$a + b \equiv a + b - n (\textrm{mod} \: n)$$이기 때문입니다. 우리는 ```n```으로 나누고 있습니다. 그렇기 때문에 나누어지는 숫자인 ```a+b```에 ```n```을 더하거나 뺀다고 해서 나머지가 변하지는 않습니다(```n```을 ```n```으로 나누면 나머지는 ```0```!). 그렇기 때문에 ```a + b - n == a - (n - b)```입니다. 굳이 이런 짓을 하는 이유는, 오버플로우를 일으키는 숫자를 일부러 작게 만들기 위해서입니다.

## 곱하기에서 오버플로우 방지하기

곱하기 역시 오버플로우에서 자유로울 수 없습니다. 두 정수를 곱한 나머지의 오버플로우를 막는 코드는 다음과 같습니다.
{% highlight C %}
// https://www.quora.com/How-can-I-execute-A-*-B-mod-C-without-overflow-if-A-and-B-are-lesser-than-C
unsigned long long mulmod(unsigned long long a, unsigned long long b, unsigned long long n)
{
    unsigned long long amod = a % n;
    unsigned long long bmod = b % n;
    
    if (amod == 0 || bmod == 0) {
        return 0;
    }
    if (amod == 1) {
        return bmod;
    }
    if (bmod == 1) {
        return amod;
    }
    
    unsigned long long asquared = mulmod(amod, bmod/2, n);
    if ((bmod & 1) == 0) {
        return addmod(asquared, asquared, n);
    }
    return addmod(amod, addmod(asquared, asquared, n), n);
}
{% endhighlight %}

이 코드는 **분할과 정복**(Divide and Conquer)을 이용하여 구현되었습니다. 우리가 구하고자 하는 것은 $$ab \,\, \textrm{mod} \,\, n$$입니다. 그런데 다음이 성립합니다.
\begin{align}
ab \,\, \textrm{mod} \,\, n = 
\begin{cases}
(ak + ak) \,\, \textrm{mod} \,\, n,& b = 2k \\\
(a + ak + ak) \,\, \textrm{mod} \,\, n,& b = 2k+1 \\\
\end{cases}
\end{align}

즉, 우리는 ```b```를 반으로 줄인 것으로 원래의 답을 구할 수 있습니다. 이렇게 되면 숫자가 작아지기 때문에 오버플로우를 막을 수 있습니다. 

## 거듭제곱의 나머지 구하기
거듭제곱의 나머지를 구하는 것도 별로 특별할 건 없습니다. 결국은 곱하기를 반복하는 것일 뿐이니까요. 그런데 $$a^{d} \,\, \textrm{mod} \,\, n$$을 단순한 반복문으로 구하면 $$d$$에 대한 시간 복잡도가 $$O(d)$$이지만, 분할과 정복을 이용한 재귀문으로 짜면 시간 복잡도를 $$O(\log d)$$로 낮출 수 있습니다.

{% highlight C %}
unsigned long long powermod(unsigned long long a, unsigned long long d, unsigned long long n)
{
    if (d == 0) {
        return 1;
    }
    unsigned long long amod = a % n;
    if (d == 1) {
        return amod;
    }
    unsigned long long h = powermod(amod, d/2, n);
    if ((d & 1) == 0) {
        return sqmod(h, n);
    }
    return mulmod(amod, sqmod(h, n), n);
}
{% endhighlight %}
