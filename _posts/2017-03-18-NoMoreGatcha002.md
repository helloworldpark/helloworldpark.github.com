---
layout: post
title:  "가챠의 폭력성 (2) - 구현편"
date:   2017-03-18 22:24:27 +0900
categories: Mathematics
---

이번 편은 가챠 뽑기의 시뮬레이션을 구현하는 파트입니다. 자세한 코드는 [이 저장소](https://github.com/helloworldpark/nomoregatcha)에서 확인할 수 있습니다.

## 시뮬레이션으로 검증해보기

문제가 어떤 것이었는지 다시 한 번 보겠습니다.

> 8개의 아이템을 다 모아야 하는데, 이 아이템은 뽑기 상자(즉, 가챠)를 하나 굴리면 하나가 나온다고 해보자.
> 이걸 다 모으려면 가챠를 평균 몇 번 돌려야 할까?

가챠의 폭력성 (1) - 이론편 에서 가챠를 뽑을 때의 확률이 같은 경우의 공식을 유도하였습니다.

\begin{align}
M_{n} &= \sum_{k=0}^{n-1} (-1)^{k} \binom{n}{k+1} \left(n-1 + \frac{n}{k+1} \right) \left(1 - \frac{k+1}{n} \right)^{n-1}  \\\
      &= 1 + n \sum_{k=1}^{n-1} \frac{1}{k}
\end{align}

이번에는 Swift를 사용하여 실제로 시뮬레이션을 해보도록 하겠습니다.

## Item 구현하기

아이템은 별 거 없습니다. 프로퍼티만 잘 들고 있어주면 됩니다.

{% highlight Swift %}
public struct Item : Hashable
{
    public var name: String
    private var _hash: Int
    public var hashValue: Int {
        get {
            return _hash
        }
    }
    ...
}
{% endhighlight %}

아이템의 구분은 아이템의 이름으로 하고, ```hashValue```는 ```String```의 해시값으로 대체합니다.

## 이상적인 GATCHA 구현하기

여기에서는 이상적인 가챠를 **뽑을 확률이 모두 동등한 것**으로 가정하겠습니다. 이번에 구현할 클래스는 이상적인 가챠로 아이템을 모을 때 평균적으로 몇 번을 뽑아야 하는지 계산해주는 녀석입니다. 그런데 막상 구현하려고 하니 조합을 구해놔야 합니다. 조합은 다음의 관계식을 만족하므로, 동적 계획법(Dynamic Programming)을 응용하면 빠르게 구할 수 있습니다.
\begin{align}
\binom{n}{k} = 
\begin{cases}
1 & \text{if } k=0,n \\\
\binom{n-1}{k-1} + \binom{n-1}{k} & \text{if } 1 \leq k < n \\\
0 & \text{if else} \\\
\end{cases}
\end{align}

조합까지 모두 구현한 클래스는 아래와 같습니다.
{% highlight Swift %}
public class IdealGatcha
{
    // Triangular array of height, width (N+1)
    private var combination: [[UInt64]]
    
    public init(N: Int)
    {
        ...
    }
    
    public func mean(_ k: Int) -> Double
    {
        var m = 0.0
        for j in 0..<k {
            let r = Double(j+1)/Double(k)
            var mp = self.bitSign(j) * Double(self.combination[k][j+1])
            mp = mp * (Double(k-1) + (1.0/r))
            mp = mp * pow(1.0 - r, Double(k-1))
            m = m + mp
        }
        return m
    }
    
    private func bitSign(_ n: Int) -> Double
    {
        return (n & 1 == 0 ? 1.0 : -1.0)
    }
    
    private func initCombination()
    {
        self.combination[1][0] = 1
        self.combination[1][1] = 1
        
        for i in 2..<(self.combination.count) {
            self.combination[i][0] = 1
            self.combination[i][i] = 1
        }
        
        for i in 2..<(self.combination.count) {
            for j in 1..<i {
                self.combination[i][j] = self.combination[i-1][j-1] + self.combination[i-1][j]
            }
        }
    }
}
{% endhighlight %}

## GATCHA 시뮬레이터

이제 가챠가 나올 확률을 조작할 수 있는 시뮬레이터를 만들어보겠습니다. 클래스 자체는 너무 기니, 중요한 메소드만 구현하면 아래와 같습니다.

{% highlight Swift %}
public struct Report
{
    public let items: Int
    public let rounds: Int
    public let min: UInt64
    public let max: UInt64
    public let mean: Double
    public let stdev: Double
    
    public func report()
}

public class Gatcha
{
    public var name: String
    public let items: [Item]
    private let discreteHelper: ERDiscrete<Item>
    private var appearedItems: Set<Item>
    
    public init(items: [Item], odds: [Double])
    
    public convenience init(odds: [Double])
    
    public func run(forRounds r: Int, maximumPick p: UInt64, reportAsFile: Bool = false) -> Report
    {
        ...
        var pickArr = [UInt64]()
        for _ in 1...r
        {
            var picks = UInt64(0)
            while self.appearedItems.count < items.count && picks < p
            {
                picks = picks + 1
                self.appearedItems.insert(self.discreteHelper.generate())
            }
            self.appearedItems.removeAll(keepingCapacity: true)
            pickArr.append(picks)
        }
        ...
    }
    ...
}
{% endhighlight %}

핵심적인 로직은 뽑은 아이템들을 보관할 ```Set```인 ```appearedItems```의 갯수가 처음 주어진 아이템의 갯수와 같아질 때까지 반복문을 돌린다는 것입니다. 그래도 한 번만 테스트하면 정이 없으니, 정해진 횟수인 ```r```만큼 테스트를 해봅니다. 난수를 생성하는 것은 제가 기존에 만들었던 난수 생성 코드를 재활용하였습니다.