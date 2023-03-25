---
layout: post
title: "【LeetCode】贪心算法"
subtitle: "[LeetCode] Greedy Algorithm"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - docker
---

## 贪心算法

贪心算法或贪心思想采用贪心的策略，保证每次操作都是**局部最优的**，从而使得最后得到的结果是**全局最优的**。

例子：

小明和小王喜欢吃苹果，小明可以吃五个，小王可以吃三个。已知苹果园里有吃不完的苹果，求小明和小王一共最多吃多少个苹果。在这个例子中，我们可以选用的贪心策略为：**每个人吃自己能吃的最多数量的苹果，这在每个人身上都是局部最优的。又因为全局结果是局部结果的简单求和，且局部结果互不相干，因此局部最优的策略也同样是全局最优的策略**。

## 相关题目

### 455. Assign Cookies (Easy)

[原题链接](https://leetcode.cn/problems/assign-cookies/)

**题目描述**

假设你是一位很棒的家长，想要给你的孩子们一些小饼干。但是，每个孩子最多只能给一块饼干。

对每个孩子 i，都有一个胃口值 g[i]，这是能让孩子们满足胃口的饼干的最小尺寸；并且每块饼干 j，都有一个尺寸 s[j] 。如果 s[j] >= g[i]，我们可以将这个饼干 j 分配给孩子 i ，这个孩子会得到满足。你的目标是尽可能满足越多数量的孩子，并输出这个最大数值。

**示例**

```
输入: g = [1,2,3], s = [1,1]
输出: 1
解释: 
你有三个孩子和两块小饼干，3个孩子的胃口值分别是：1,2,3。
虽然你有两块小饼干，由于他们的尺寸都是1，你只能让胃口值是1的孩子满足。
所以你应该输出1。
```

```
输入: g = [1,2], s = [1,2,3]
输出: 2
解释: 
你有两个孩子和三块小饼干，2个孩子的胃口值分别是1,2。
你拥有的饼干数量和尺寸都足以让所有孩子满足。
所以你应该输出2.
```

**题解**

首先我们使用`sort函数（头文件是algorithm）`对孩子的胃口值和饼干的尺寸从小到大进行排序，利用**贪心的思想，我们先让胃口最小的孩子吃饱（即选择饼干大小大于等于其胃口中最小的那一个）**，满足这个孩子后，剩余孩子同样采用这个策略，直到孩子或者饼干遍历完了。

```c++
class Solution {
    public:
        int findContentChildren(vector<int>& g, vector<int>& s) {
            sort(g.begin(), g.end());
            sort(s.begin(), s.end());
            int child = 0, cookie = 0;
            while(child < g.size() && cookie < s.size()){
                if(g[child] <= s[cookie])
                    child++;
                cookie++;
            }
            return child;
        }
};
```

### 135. Candy (Hard)

[原题链接](https://leetcode.cn/problems/candy/)

**题目描述**

n 个孩子站成一排。给你一个整数数组 ratings 表示每个孩子的评分。

你需要按照以下要求，给这些孩子分发糖果：

每个孩子至少分配到 1 个糖果。
相邻两个孩子评分更高的孩子会获得更多的糖果。
请你给每个孩子分发糖果，计算并返回需要准备的最少糖果数目 。

**示例**

```
输入：ratings = [1,0,2]
输出：5
解释：你可以分别给第一个、第二个、第三个孩子分发 2、1、2 颗糖果。
```

```
输入：ratings = [1,2,2]
输出：4
解释：你可以分别给第一个、第二个、第三个孩子分发 1、2、1 颗糖果。
     第三个孩子只得到 1 颗糖果，这满足题面中的两个条件。
```

**题解**

题目要求相邻两个孩子中评分更高的孩子会获得更多的糖果，因此我们可以**采用两次遍历，一次从左到右，一次从右到左，每次比较只需要满足两个孩子中糖果更多即可，实现局部最优，最终使得所有的孩子都满足这个条件**。

>- 由于ratings.length可能等于1，那么rating[1]就越界了，因此为了程序的鲁棒性，在开始先对ratings的大小进行判断
>- “所有至少分配到一个糖果”，可以直接创建一个ratings.size()大小，初始值均为1的vector容器
>- “相邻两个孩子评分更高的孩子会获得更多的糖果”，不能天真的将评分高的孩子的糖果数量设置为小的那个孩子的糖果数量+1，还需要和当前的糖果数量相比较，选择大的那个值（考虑相邻两边）
>- 可以通过`accumulate函数（头文件是numeric）`方便的求解容器的和，[accumulate函数介绍](https://blog.csdn.net/Jeanphorn/article/details/45114233)

```c++
class Solution {
    public:
        int candy(vector<int>& ratings) {
            int size = ratings.size();
            if(size < 2)
                return size;
            vector<int> candies(size, 1);
            for(int i = 1; i < size; i++){
                if(ratings[i] > ratings[i-1])
                    candies[i] = max(candies[i-1] + 1, candies[i]);
            }
            for(int i = size - 1; i > 0; i--){
                if(ratings[i-1] > ratings[i])
                    candies[i-1] = max(candies[i] + 1, candies[i-1]);
            }
            return accumulate(candies.begin(), candies.end(), 0);
        }
};
```

### 435. Non-overlapping Intervals (Medium)

[原题链接](https://leetcode.cn/problems/non-overlapping-intervals/)

**题目描述**

给定一个区间的集合 intervals ，其中 intervals[i] = [starti, endi] 。返回需要移除区间的最小数量，使剩余区间互不重叠（起止相连不算重叠） 。

**示例**

```
输入: intervals = [[1,2],[2,3],[3,4],[1,3]]
输出: 1
解释: 移除 [1,3] 后，剩下的区间没有重叠。
```

```
输入: intervals = [ [1,2], [1,2], [1,2] ]
输出: 2
解释: 你需要移除两个 [1,2] 来使剩下的区间没有重叠。
```

```
输入: intervals = [ [1,2], [2,3] ]
输出: 0
解释: 你不需要移除任何区间，因为它们已经是无重叠的了。
```

**题解**

要求移除区间最小，使得剩余区间互不重叠，等价于尽量多保留不重叠的区间。如何用代码实现呢？**选择保留区间在这题中就极其关键，试想结尾为1的区间和结尾为7的区间哪个更容易与其它区间重叠呢？很明显当然是结尾更大的那个区间，因此我们需要优先保留结尾小且不相交的区间**。

>- 使用lambda函数和sort函数对intervals进行排序，优先按照结尾从小到大排序，结尾相等则按开头从小到大排序
>- 区间之间的比较不是与前一个小区间比较，而是与前一个不重叠的大区间比较，这样才不会重叠，因此在比较后需要更新当前不重叠区间的结尾值
>  1. 两个区间重叠，结尾值选择小的那个
>  2. 两个区间不重叠，结尾值选择大的那个

```c++
class Solution {
    public:
        int eraseOverlapIntervals(vector<vector<int>>& intervals) {
            int size = intervals.size();
            sort(intervals.begin(), intervals.end(), [](vector<int>& a, vector<int>& b){
                return a[1] < b[1] || (a[1] == b[1] && a[0] < b[0]);
            });
            int removed = 0, pre = intervals[0][1];
            for(int i = 1; i < size; i++){
                if(intervals[i][0] < pre)
                    removed++;
                else
                    pre = intervals[i][1];
            }
            return removed;
        }
};
```

### 605. Can Place Flowers (Easy)

[原题链接](https://leetcode.cn/problems/can-place-flowers/)

**题目描述**

假设有一个很长的花坛，一部分地块种植了花，另一部分却没有。可是，花不能种植在相邻的地块上，它们会争夺水源，两者都会死去。

给你一个整数数组  flowerbed 表示花坛，由若干 0 和 1 组成，其中 0 表示没种植花，1 表示种植了花。另有一个数 n ，能否在不打破种植规则的情况下种入 n 朵花？能则返回 true ，不能则返回 false。

**示例**

```
输入：flowerbed = [1,0,0,0,1], n = 1
输出：true
```

```
输入：flowerbed = [1,0,0,0,1], n = 2
输出：false
```

**题解**

采用**贪心的思想，当前位置能种花就种花**。

> 当前位置可以种花有三种情况：
>
> - 最左边的位置，只需要右边位置没种花即可
> - 最右边的位置，只需要左边位置没种花即可
> - 中间的位置，需要左右两边位置都没种花
>
> if语句中 `||` 的顺序很关键，`||` 的特点是只要前一个条件为真就不判断后面的条件了，因此i == flowerbed.size() - 1和 !i 两个条件需要在前面，这样就不会出现越界的情况

```c++
class Solution {
    public:
        bool canPlaceFlowers(vector<int>& flowerbed, int n) {
            int flag = 0;
            for(int i = 0; i < flowerbed.size(); i++){
                if(!flowerbed[i] && (i == flowerbed.size() -1 || !flowerbed[i+1]) && (!i || !flowerbed[i-1])){
                    flowerbed[i] = 1;
                    flag++;
                }
            }
            return flag >= n;
    }
};
```

### 452. Minimum Number of Arrows to Burst Balloons (Medium)

[原题链接](https://leetcode.cn/problems/minimum-number-of-arrows-to-burst-balloons/)

**题目描述**

有一些球形气球贴在一堵用 XY 平面表示的墙面上。墙面上的气球记录在整数数组 points ，其中points[i] = [xstart, xend] 表示水平直径在 xstart 和 xend之间的气球。你不知道气球的确切 y 坐标。

一支弓箭可以沿着 x 轴从不同点完全垂直地射出。在坐标 x 处射出一支箭，若有一个气球的直径的开始和结束坐标为 xstart，xend， 且满足  xstart ≤ x ≤ xend，则该气球会被引爆 。可以射出的弓箭的数量没有限制 。 弓箭一旦被射出之后，可以无限地前进。

给你一个数组 points ，返回引爆所有气球所必须射出的最小弓箭数 。

**示例**

```
输入：points = [[10,16],[2,8],[1,6],[7,12]]
输出：2
解释：气球可以用2支箭来爆破:
-在x = 6处射出箭，击破气球[2,8]和[1,6]。
-在x = 11处发射箭，击破气球[10,16]和[7,12]。
```

```
输入：points = [[1,2],[3,4],[5,6],[7,8]]
输出：4
解释：每个气球需要射出一支箭，总共需要4支箭。
```

```
输入：points = [[1,2],[2,3],[3,4],[4,5]]
输出：2
解释：气球可以用2支箭来爆破:
- 在x = 2处发射箭，击破气球[1,2]和[2,3]。
- 在x = 4处射出箭，击破气球[3,4]和[4,5]。
```

**题解**

思路和435题类似，但这个题是尽可能使得多个区间重叠，同样也是先进行排序（按照区间结尾从小到大排，若区间结尾相等则按区间开头从小到大排），然后再依次判断是否能加入前一个重叠的区间，注意重叠区间的临界值更新。

>- 若当前区间可以加入前一个重叠区间，则重叠区间的临界值更新为区间中最小的结尾值
>- 若当前区间不可以加入前一个重叠区间，则该区间创建一个新的重叠区间

```c++
class Solution {
    public:
        int findMinArrowShots(vector<vector<int>>& points) {
            if(points.size() < 2)
                return points.size();
            sort(points.begin(), points.end(), [](const vector<int>& a, const vector<int>& b){
                return a[1] < b[1] || (a[1] == b[1] && a[0] < b[0]);
            });
            int pre = points[0][1], num = 1;
            for(int i = 1; i < points.size(); i++){
                if(points[i][0] <= pre)
                    pre = min(pre, points[i][1]);
                else{
                    pre = points[i][1];
                    num ++;
                }
            }
            return num;
    }
};
```

### 763. Partition Labels (Medium)

[原题链接](https://leetcode.cn/problems/partition-labels/)

**题目描述**

给你一个字符串 s 。我们要把这个字符串划分为尽可能多的片段，同一字母最多出现在一个片段中。

注意，划分结果需要满足：将所有划分结果按顺序连接，得到的字符串仍然是 s 。

返回一个表示每个字符串片段的长度的列表。

**示例**

```
输入：s = "ababcbacadefegdehijhklij"
输出：[9,7,8]
解释：
划分结果为 "ababcbaca"、"defegde"、"hijhklij" 。
每个字母最多出现在一个片段中。
像 "ababcbacadefegde", "hijhklij" 这样的划分是错误的，因为划分的片段数较少。 
```

```
输入：s = "eccbbbbdec"
输出：[10]
```

**题解**

这个题其实也是一个区间问题，要求将字符串划分为尽可能多的片段，不加其他条件的话有很多种划分手段，当然每个字母都划分一次肯定最后的片段最多。但要求**同一字母出现在一个片段中**就很局限了，这也是解题的关键。**拿示例1看，划分结果为 "ababcbaca"、"defegde"、"hijhklij"，可以发现所有小片段中任何一个字母的最后出现位置都小于等于片段最后一个字母出现的位置**。

> 因此我们可以先对字符串进行一次遍历，得到字符串中每个字母出现的最后位置。再对字符串进行一次遍历：
>
> - 当前字母出现的最后位置若小于等于已划分区间的最后位置，则表示当前字母可以加入已划分区间，不进行操作
> - 当前字母出现的最后位置若大于已划分区间的最后位置（并且字母当前的位置小于已划分区间的最后位置），则表示需要扩大已划分区间，更新已划分区间的最后位置（选择大的那个）
> - 若到达已划分区间的最后位置，则表示该片段结束，更新下一个需要划分区间开始位置

```c++
class Solution {
    public:
       vector<int> partitionLabels(string s) {
            vector<int> alphabet(26);
            for(int i = 0; i < s.size(); i++)
                alphabet[s[i] - 'a'] = i;
            vector<int> ans;
            int begin =0, end = 0;
            for(int i = 0; i < s.size(); i++){
                end = max(alphabet[s[i] - 'a'], end);
                if(i == end){
                    ans.push_back(end - begin + 1);
                    begin = end + 1;
                }
            }
            return ans;
    }
};
```

### 122. Best Time to Buy and Sell Stock Ⅱ (Medium)

[原题链接](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-ii/)

**题目描述**

给你一个整数数组 prices ，其中 prices[i] 表示某支股票第 i 天的价格。

在每一天，你可以决定是否购买和/或出售股票。你在任何时候最多只能持有一股股票。你也可以先购买，然后在同一天出售。

返回你能获得的最大利润。

**示例**

```
输入：prices = [7,1,5,3,6,4]
输出：7
解释：在第 2 天（股票价格 = 1）的时候买入，在第 3 天（股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5 - 1 = 4 。
     随后，在第 4 天（股票价格 = 3）的时候买入，在第 5 天（股票价格 = 6）的时候卖出, 这笔交易所能获得利润 = 6 - 3 = 3 。
     总利润为 4 + 3 = 7 。
```

```
输入：prices = [1,2,3,4,5]
输出：4
解释：在第 1 天（股票价格 = 1）的时候买入，在第 5 天 （股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5 - 1 = 4 。
     总利润为 4 。
```

```
输入：prices = [7,6,4,3,1]
输出：0
解释：在这种情况下, 交易无法获得正利润，所以不参与交易可以获得最大利润，最大利润为 0 。
```

**题解**

这个题一定不要想太复杂，不要去思考到底哪天买哪天卖，怎么个卖法，我们只需要满足总利润最大就行了。同样采用贪心的思想，**相邻两天能卖就卖（有利润就卖），实现局部最优，最后实现全局最优**。

```c++
class Solution {
    public:
        int maxProfit(vector<int>& prices) {
            if(prices.size() < 2)
                return 0;
            int sum = 0;
            for(int i = 1; i < prices.size(); i++){
                if(prices[i] > prices[i - 1])
                    sum += prices[i] - prices[i - 1];
            }
            return sum;
    }
};
```

### 406. Queue Reconstruction by Height (Medium)

[原题链接](https://leetcode.cn/problems/queue-reconstruction-by-height/)

**题目描述**

假设有打乱顺序的一群人站成一个队列，数组 people 表示队列中一些人的属性（不一定按顺序）。每个 people[i] = [hi, ki] 表示第 i 个人的身高为 hi ，前面正好有 ki 个身高大于或等于 hi 的人。

请你重新构造并返回输入数组 people 所表示的队列。返回的队列应该格式化为数组 queue ，其中 queue[j] = [hj, kj] 是队列中第 j 个人的属性（queue[0] 是排在队列前面的人）。

**示例**

```
输入：people = [[7,0],[4,4],[7,1],[5,0],[6,1],[5,2]]
输出：[[5,0],[7,0],[5,2],[6,1],[4,4],[7,1]]
解释：
编号为 0 的人身高为 5 ，没有身高更高或者相同的人排在他前面。
编号为 1 的人身高为 7 ，没有身高更高或者相同的人排在他前面。
编号为 2 的人身高为 5 ，有 2 个身高更高或者相同的人排在他前面，即编号为 0 和 1 的人。
编号为 3 的人身高为 6 ，有 1 个身高更高或者相同的人排在他前面，即编号为 1 的人。
编号为 4 的人身高为 4 ，有 4 个身高更高或者相同的人排在他前面，即编号为 0、1、2、3 的人。
编号为 5 的人身高为 7 ，有 1 个身高更高或者相同的人排在他前面，即编号为 1 的人。
因此 [[5,0],[7,0],[5,2],[6,1],[4,4],[7,1]] 是重新构造后的队列。
```

```
输入：people = [[6,0],[5,0],[4,0],[3,2],[2,2],[1,4]]
输出：[[4,0],[5,0],[2,2],[3,2],[1,4],[6,0]]
```

**题解**

这个题的关键在于排序，排序排好了，这题已经做完了90%。由于k表示前面正好有 k个身高大于或等于 h 的人，因此我们首先按身高由高到低排，如果身高相等则按k从小到大排，排序好了插入就好了。

```c++
class Solution {
    public:
        vector<vector<int>> reconstructQueue(vector<vector<int>>& people) {
            sort(people.begin(), people.end(), [](const vector<int>& u, const vector<int>& v) {
                return u[0] > v[0] || (u[0] == v[0] && u[1] < v[1]);
            });
            vector<vector<int>> ans;
            for (auto& person: people) {
                ans.insert(ans.begin() + person[1], person);
            }
            return ans;
        }
};
```

### 665. Non-decreasing Array (Medium)

[原题链接](https://leetcode.cn/problems/non-decreasing-array/)

**题目描述**

给你一个长度为 n 的整数数组 nums ，请你判断在最多改变1 个元素的情况下，该数组能否变成一个非递减数列。

我们是这样定义一个非递减数列的： 对于数组中任意的 i (0 <= i <= n-2)，总满足 nums[i] <= nums[i + 1]。

**示例**

```
输入: nums = [4,2,3]
输出: true
解释: 你可以通过把第一个 4 变成 1 来使得它成为一个非递减数列。
```

```
输入: nums = [4,2,1]
输出: false
解释: 你不能在只改变一个元素的情况下将其变为非递减数列。
```

**题解**

一开始我的想法比较简单：相邻两数两两比较，只要出现两次非递减的情况就返回false。提交通过了320 / 335个测试用例，最后一个输入是[3,4,2,3]，这个输入如果按照一开始的思路则只有一次非递减的情况，但很明显是不可能只修改一个元素使得数组变成非递减数列。

问题出在我们比较发现出现非递减情况后，**没有对元素进行修改而是接着用这个元素和下一个元素进行比较**。怎么更改元素呢？看下面三个例子：

1. 4，2，3

2. -1，4，2，3

3. 2，3，3，2，4

我们通过分析上面三个例子可以发现，当我们发现后面的数字小于前面的数字产生冲突后：

- 有时候需要修改前面那个数字(比如前两个例子需要修改4)，


- 有时候却要修改后面那个数字(比如前第三个例子需要修改2)，


那么有什么内在规律吗？是有的，**判断修改那个数字其实跟再前面一个数的大小有关系，首先如果再前面的数不存在，比如例子1，4前面没有数字了，我们直接修改前面的数字为当前的数字即可。而当再前面的数字存在，并且小于当前数时，比如例子2，-1小于2，我们还是需要修改前面的数字为当前数字；如果再前面的数大于当前数，比如例子3，3大于2，我们需要修改当前数为前面的数**。

> 写条件的时候注意可能会发生越界，比如以下写法就是不对的：
>
> -  `nums[i] >= nums[i-2] || i == 1`
> - `nums[i] < nums[i-2] && i != 1`

```c++
class Solution {
    public:
        bool checkPossibility(vector<int>& nums) {
            if(nums.size() < 2)
                return true;
            int count = 0;
            for(int i = 1; i < nums.size() && count != 2; i++){
                if(nums[i] < nums[i-1]){
                    if(i == 1 || nums[i] >= nums[i-2]){
                        nums[i-1] = nums[i];
                    }
                    else
                        nums[i] = nums[i-1];
                    count++;
                }
            }
            if(count == 2)
                return false;
            return true;
    }
};
```

## 参考资料

[1] [LeetCode 101：和你一起你轻松刷题（C++）](https://github.com/changgyhub/leetcode_101/)