# Approximate online quantile calculations. Greenwald-Kanna algorithm.

The article ["A Survey of Approximate Quantile Computation on Large-Scale Data"](https://arxiv.org/pdf/2004.08255.pdf) describes several algorithms on calculating quantiles over streaming or distributed data. I will implement the Greenwald - Khanna algorithm (referenced as GK01 in the article). This is an algorithm for calculating the approximate quantiles over a streaming data (data that can be added at any time). 

Recall: an approximate $\phi$ - quantile with an an $\epsilon$ error is any observation from data that belongs to $[r - \epsilon n; r + \epsilon n]$, where $r = \lfloor\phi n \rfloor$.

## Description of an algorithm

I implemented the Greenwald-Khanna algorithm that is mentioned as GK01 in the assignment paper.

Firstly, let us denote $r_{min}(v)$ and $r_{max}(v)$ as respectively lower and upper bounds on the ranks of $v$ among
the $n$ observations got so far. 

The algorithm introduces the new data structure named `Summary`. It consists of the tuples $t_i$ = $(v_i, g_i, \Delta_i)$, where $v_i$ - a value that corresponds to one
of the elements in the data sequence seen thus far,  $g_i$ = $r_{min}(v_i) - r_{min}(v_{i-1})$ and $\Delta_i$ = $r_{max}(v_{i}) - r_{min}(v_{i})$. The tuples are always kept sorted by $v_i$ in ascending order. Note that $v_0$ and $v_{s-1}$ represent the minimum and maximum values of data, where $s$ - number of tuples stored.

It easy to prove by simple substitution that $r_{min} (v_i) = \sum_{j=0}^i g_j$ and $r_{max} (v_i) = \sum_{j=0}^i g_j + \Delta_i$. So, the tuple $t_i$ stores the bounds on ranks that $v_i$ represents.

As it is proven in the original paper of algorithm ([Link](http://infolab.stanford.edu/~datar/courses/cs361a/papers/quantiles.pdf) pg. 60 Corollary 1), given the Summary `S` at any time $n$, if all for all $t_i$ we hold $g_i + \Delta_i \le 2\epsilon n$, we can answer any $\phi$ - quantile query with $\epsilon$ precision. In other words, to satisfy the approximation asuumption with $\epsilon$ error, at every time we need to keep tuples in such way that we cover all values of original data set and keep $v_i \in [r - \epsilon n; r + \epsilon n]$, where $r = \lfloor\phi n \rfloor$.

Now, let's go to the algorithm itself.

#### Quantile operation

The main thing we want to get is the $\phi$ quantile. Firstly, we need to calculate $r = \lfloor\phi n \rfloor$. Secondly, we need to check for border cases:
- If we get $r = 0$, we just return the first element (minimum) $v_0$ by the definition of zero-quantile.
- If we get $\phi = 1$, we just return the last element (maximum) $v_{s-1}$.
- If it appears that the upper bound of approximation $r + \epsilon n \ge n$, we return the last element as well because the approximation conditions hold.

In other cases we proceed the following strategy. We find (by iterating through tuples) the smallest $i$ such that for $t_i$: $r_{max}(v_i) \gt r + \epsilon n$, which means that this tuple may violate the approximation criteria. Then, we return the value $v_{i-1}$.

#### Insertion operation

To insert a value $v$ into our `Summary`, we need to find $i$ such that $v_{i-1} \le v \lt v_i$. Then we insert a tuple $(v, 1, g_i + \Delta_i - 1)$ between $t_{i-1}$ and $t_i$. If $v$ is the new minimum or maximum instead of $(v, 1, g_i + \Delta_i - 1)$ we insert $(v,1,0)$ in the beggining or the end respectively. Doing insertion in such way we always hold the structure correct.

#### Compression

In order to decrease space taken by `Summary` before every $\frac 1{2\epsilon}$ insertion we compress the data.

The algorithm for compression is the following. By taking $i$ as last index of the tuples list, we find the longest subarray $t_j, ..., t_i$ such that $t_j + ... + t_i + \Delta_i \lt 2\epsilon n$. If this subarray contains more than one tuple ($j \lt i$), we replace $t_j, ..., t_i$ by new tuple $(v_i,t_j + ... + t_i,\Delta_i)$. Then we continue doing the same for $i = j - 1$ until $i$ reaches 1. By doing that we merge tuples with "low range" into tuples with "larger range" without violating the criterias with the benefit of freeing up space.

## Comparison of performance

Below, we will see how the approximate algorithm works in comparison with the numpy implementation. We will watch on memory and time consumtion.

### Generating data and calculating performance

Here, we iterate for `iterations` times and on each one (let`s call it timestamp) generate the random data, put it into two algorythms (Approximate and numpy) and then calculating the random quantile on both algorithms with measuring and storing performances for further plotting.

![perf comp](https://github.com/seoful/greenwald-khanna-quantile-algo/assets/34482712/b44cf831-ed08-4181-a7b0-d42c0e6e2aa4)

It is obviously seen that numpy algorithm has at average linear time of calculating quantiles while approximate algorithm has something that seems like a constant. Though, because of nearence to the 0 value, we cannot make this conclusion. For it, we need to process more data but it it time consuming. But, we can say that approximate algorithm has a much better time performance calculating quantiles.

![n](https://github.com/seoful/greenwald-khanna-quantile-algo/assets/34482712/161970a4-712b-487b-a8d4-ae3d027f85f6)

We can see that approximate algorithm is a lot better by memory consumption than a numpy, as it does not store all the data it got.

## Test

Here we will show that the approximate algorithm is working properly.

We will take some distribution (e.g normal) and see how the calculated by our algorithm quantiles behaves with the increase in sample size and whether they tend to approximate more and more the quantile function (`inverse cdf`) of the distribution.

#### Normal distribution
![normal](https://github.com/seoful/greenwald-khanna-quantile-algo/assets/34482712/9a877c23-ae18-489c-9cfb-80dab6f08acd)

#### Exponential distribution
![expon](https://github.com/seoful/greenwald-khanna-quantile-algo/assets/34482712/74d18a76-e166-4b93-af4f-385c0efa53da)

#### Logistic distribution
![logist](https://github.com/seoful/greenwald-khanna-quantile-algo/assets/34482712/3310880b-fc55-432b-9e33-89349688b5fb)

#### Poisson distribution
![poisson](https://github.com/seoful/greenwald-khanna-quantile-algo/assets/34482712/f68ce028-74c6-42c6-93b1-448b580266b4)




