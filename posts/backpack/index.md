# 背包问题golang


最近的工作都跟集群调度有关，一直在为了满足用户需求添加各种调度策略，现在也暂时告一段落了，抽时间总结思考了之前的工作，调度本质上就是背包问题，但是相当复杂，涉及到多维多重背包、组合背包、依赖背包等。又重新开始学习背包问题，这里先从简单的01背包开始讲，在网上也找到了很多相关的文章，但是很遗憾，我找到的很多关于用golang实现背包的文章中给出的代码都是有问题的，后决定自己写出来，也希望大家一起思考，不要被误导了。

### 题目

有有N件物品和一个容量为V的背包。第i件物品的费用是c[i]，价值是w[i]。求解将哪些物品装入背包可使价值总和最大。

### 基本思想

这是最基础的背包问题，特点是：每种物品仅有一件，可以选择放或不放。

用子问题定义状态：即f\[i][v]表示前i件物品恰放入一个容量为v的背包可以获得的最大价值。则其状态转移方程便是：

> f\[i][v]=max{f\[i-1][v],f\[i-1][v-c[i]]+w[i]}

这个方程非常重要，基本上所有跟背包相关的问题的方程都是由它衍生出来的。所以有必要将它详细解释一下：“将前i件物品放入容量为v的背包中”这个子问题，若只考虑第i件物品的策略（放或不放），那么就可以转化为一个只牵扯前i-1件物品的问题。如果不放第i件物品，那么问题就转化为“前i-1件物品放入容量为v的背包中”，价值为f\[i-1][v]；如果放第i件物品，那么问题就转化为“前i-1件物品放入剩下的容量为v-c[i]的背包中”，此时能获得的最大价值就是f\[i-1][v-c[i]]再加上通过放入第i件物品获得的价值w[i]。

### 代码实现

#### 测试用例

```go
func main() {
	v := []int{7, 5, 8} //物品大小
	w := []int{2, 3, 4} //物品价值
	goods := [][]int{
		[]int{7, 2},
		[]int{5, 3},
		[]int{8, 4},
	}
	fmt.Println(zeroonepack1(v, w, 5))
	fmt.Println(zeroonepack(goods, 5))
	fmt.Println(zeroonepack2(v, w, 5))
}

// 结果应该都是3才对
```

#### 二维数组实现

```go
// 0 消耗 1 价值
// i 物品数 j 容量
// dp[i][j] = max(dp[i-1][j], goods[i][1] + dp[i-1][j-goods[i][0]])
func zeroonepack(goods [][]int, capacity int) int {
	num := len(goods)
	dp := make([][]int, num)
	for i := 0; i < num; i++ {
		dp[i] = make([]int, capacity+1)
	}

	for i := goods[0][0]; i < capacity+1; i++ {
		dp[0][i] = goods[0][1]
	}

	for i := 1; i < num; i++ {
    // j不能直接从good[i][0]开始
		for j := 0; j <= capacity; j++ {
      // 此if else判断必须要有
			if j >= goods[i][0] {
				dp[i][j] = max(dp[i-1][j], dp[i-1][j-goods[i][0]]+goods[i][1])
			} else {
				dp[i][j] = dp[i-1][j]
			}
		}
	}

	return dp[num-1][capacity]
}

func zeroonepack1(weight []int, value []int, capacity int) int {
	num := len(weight)
	dp := make([][]int, num)
	for i := 0; i < num; i++ {
		dp[i] = make([]int, capacity+1)
	}

	for i := weight[0]; i <= capacity; i++ {
		dp[0][i] = value[0]
	}

	for i := 1; i < num; i++ {
		for j := 0; j <= capacity; j++ {
			if j >= weight[i] {
				dp[i][j] = max(dp[i-1][j], dp[i-1][j-weight[i]]+value[i])
			} else {
				dp[i][j] = dp[i-1][j]
			}
		}
	}

	return dp[num-1][capacity]
}
```

上面都是二维数组的实现，空间复杂度是O(VN)，区别就是传入的参数不同，用二维数组表示物品，或者用两个一维数组分别表示物品的消耗（重量或体积等）和价值，特别注意上面的注释，很多golang版本的代码就是那里出的问题，错误版本的代码参考[这里](https://studygolang.com/articles/11459)

#### 一维数组实现

以上方法的时间和空间复杂度均为O(VN)，其中时间复杂度应该已经不能再优化了，但空间复杂度却可以优化到O。

先考虑上面讲的基本思路如何实现，肯定是有一个主循环i=1..N，每次算出来二维数组f[i][0..V]的所有值。那么，如果只用一个数组f[0..V]，能不能保证第i次循环结束后f[v]中表示的就是我们定义的状态f\[i][v]呢？f\[i][v]是由f\[i-1][v]和f\[i-1][v-c[i]]两个子问题递推而来，能否保证在推f\[i][v]时（也即在第i次主循环中推f[v]时）能够得到f\[i-1][v]和f\[i-1][v-c[i]]的值呢？事实上，这要求在每次主循环中我们以v=V..0的顺序推f[v]，这样才能保证推f[v]时f[v-c[i]]保存的是状态f\[i-1][v-c[i]]的值。代码如下：

```go
func zeroonepack2(weight []int, value []int, capacity int) int {
	num := len(weight)
	dp := make([]int, capacity+1)
	for i := weight[0]; i < capacity; i++ {
		dp[i] = value[0]
	}

	for i := 1; i < num; i++ {
		for j := capacity; j >= 0; j-- {
			if j >= weight[i] {
				dp[j] = max(dp[j], dp[j-weight[i]]+value[i])
			}
		}
	}

	return dp[capacity]
}
```

其中的f[v]=max{f[v],f[v-c[i]]}一句恰就相当于我们的转移方程`f[i][v]=max{f[i-1][v],f[i-1][v-c[i]]}`，因为现在的f[v-c[i]]就相当于原来的f\[i-1][v-c[i]]。如果将v的循环顺序从上面的逆序改成顺序的话，那么则成了f\[i][v]由f\[i][v-c[i]]推知，与本题意不符，但它却是另一个重要的背包问题最简捷的解决方案，故学习只用一维数组解01背包问题是十分必要的。

