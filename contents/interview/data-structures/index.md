# LeetCode

## 1. 数组与双指针 (8 题)

这部分是所有面试的基石，主要考查对边界条件的控制和空间优化。

### Merge Sorted Array: 逆向双指针的经典应用。

将两个有序数组合并到第一个数组中，利用第一个数组末尾的空余空间，从后往前比较并填充最大值，以避免覆盖。

```javascript
/**
 * @param {number[]} nums1 - 目标数组，包含 m 个有效元素，总长度为 m + n
 * @param {number} m - nums1 中的有效元素数量
 * @param {number[]} nums2 - 源数组，长度为 n
 * @param {number} n - nums2 中的元素数量
 */
var merge = function (nums1, m, nums2, n) {
  // 设置三个指针：
  // i 指向 nums1 有效元素的末尾
  // j 指向 nums2 的末尾
  // k 指向 nums1 最终容器的最末尾
  let i = m - 1;
  let j = n - 1;
  let k = m + n - 1;

  // 当 nums2 中还有元素没被合并时继续循环
  while (j >= 0) {
    // 如果 nums1 还有剩余元素且 nums1 当前元素大于 nums2 当前元素
    if (i >= 0 && nums1[i] > nums2[j]) {
      // 将较大的 nums1[i] 放到最后，并移动指针
      nums1[k] = nums1[i];
      i--;
    } else {
      // 否则将较大的 nums2[j]（或 nums1 已耗尽时剩余的 nums2）放到最后
      nums1[k] = nums2[j];
      j--;
    }
    // 每次填充完一个位置，容器末尾指针前移
    k--;
  }

  // 注意：不需要处理 i >= 0 的情况，因为如果 nums2 先耗尽，
  // nums1 剩下的元素本来就在正确的位置上。
};
```

### Remove Duplicates from Sorted Array II: 快慢指针控制重复次数。

利用数组已排序的特性，使用快慢双指针遍历，只有当快指针遇到与慢指针不同的新元素时，才将其移动到慢指针的下一个位置。

```javascript
/**
 * @param {number[]} nums - 升序排列的整数数组
 * @return {number} - 返回唯一元素的个数 k
 */
var removeDuplicates = function (nums) {
  // 如果数组为空，直接返回 0
  if (nums.length === 0) return 0;

  // k 是慢指针，指向当前已确定的唯一元素序列的末尾
  // 因为数组是有序的，第一个元素 nums[0] 必然是唯一的，所以从索引 0 开始
  let k = 0;

  // i 是快指针，从第二个元素开始遍历数组
  for (let i = 1; i < nums.length; i++) {
    // 如果快指针指向的元素与慢指针指向的元素不同
    // 说明找到了一个新的唯一元素
    if (nums[i] !== nums[k]) {
      // 慢指针前移一位，并将新元素复制到该位置
      k++;
      nums[k] = nums[i];
    }
  }

  // 返回唯一元素的个数，即索引 k + 1
  return k + 1;
};
```

### Rotate Array: 三次翻转法的巧妙空间优化。

通过三次反转数组的技巧实现原地旋转：先反转整个数组，再分别反转前 $k$ 个元素和剩余的元素。

```javascript
/**
 * @param {number[]} nums - 待旋转的数组
 * @param {number} k - 旋转的步数
 * @return {void} - 直接修改原数组，不返回任何值
 */
var rotate = function (nums, k) {
  const n = nums.length;
  // 1. 处理 k 大于数组长度的情况，取余数即可
  k %= n;

  // 辅助函数：原地反转数组中指定范围的元素
  const reverse = (arr, start, end) => {
    while (start < end) {
      // 解构赋值进行元素交换
      [arr[start], arr[end]] = [arr[end], arr[start]];
      start++;
      end--;
    }
  };

  // 2. 核心步骤：
  // 第一步：反转整个数组
  // 结果示例: [7,6,5,4,3,2,1]
  reverse(nums, 0, n - 1);

  // 第二步：反转前 k 个元素
  // 结果示例: [5,6,7,4,3,2,1]
  reverse(nums, 0, k - 1);

  // 第三步：反转剩余的元素
  // 结果示例: [5,6,7,1,2,3,4]
  reverse(nums, k, n - 1);
};
```

### Best Time to Buy and Sell Stock: 贪心思想的入门。

遍历数组并动态更新历史最低价格，同时计算每一天卖出所能获得的最大利润，取其中的最大值。

```javascript
/**
 * @param {number[]} prices - 每日股票价格数组
 * @return {number} - 能够获得的最大利润
 */
var maxProfit = function (prices) {
  // minPrice 记录遍历到目前为止的最低买入价格
  let minPrice = Infinity;
  // maxProfit 记录遍历到目前为止能获得的最大利润
  let maxProfit = 0;

  for (let i = 0; i < prices.length; i++) {
    // 如果当前价格比历史最低价还低，更新最低价（寻找潜在的买入点）
    if (prices[i] < minPrice) {
      minPrice = prices[i];
    }
    // 否则计算“如果今天卖出”能赚多少钱，并更新全局最大利润
    else if (prices[i] - minPrice > maxProfit) {
      maxProfit = prices[i] - minPrice;
    }
  }

  return maxProfit;
};
```

### Trapping Rain Water: (高频 Hard) 双指针或单调栈的巅峰之作。

使用双指针法从两端向中间移动，实时维护左侧和右侧的最高边界，每个位置能接的水量取决于它左右两侧最高边界中的较小值。

```javascript
/**
 * @param {number[]} height - 海拔高度数组
 * @return {number} - 总接水量
 */
var trap = function (height) {
  // left 和 right 分别指向数组的两端
  let left = 0;
  let right = height.length - 1;

  // leftMax 和 rightMax 记录左右两边遍历过的最高高度
  let leftMax = 0;
  let rightMax = 0;

  let totalWater = 0;

  while (left < right) {
    // 如果左边的柱子比右边低
    if (height[left] < height[right]) {
      // 如果当前高度大于等于左侧最大高度，无法接水，更新最大高度
      if (height[left] >= leftMax) {
        leftMax = height[left];
      } else {
        // 当前高度小于左侧最大，且由于左侧高度 < 右侧高度，
        // 说明右侧一定存在比 leftMax 更高的边界，水坑深度由 leftMax 决定
        totalWater += leftMax - height[left];
      }
      left++;
    }
    // 如果右边的柱子比左边低（或相等）
    else {
      if (height[right] >= rightMax) {
        rightMax = height[right];
      } else {
        // 同理，左侧一定存在比 rightMax 更高的边界
        totalWater += rightMax - height[right];
      }
      right--;
    }
  }

  return totalWater;
};
```

### Two Sum II - Input Array Is Sorted: 有序数组双指针的模板题。

由于数组已经是升序排列的，我们可以使用双指针法，分别从数组的首尾向中间移动。这种方法可以利用排序特性，在 $O(n)$ 时间复杂度和 $O(1)$ 额外空间内找到目标值。

```javascript
/**
 * @param {number[]} numbers - 已排序的 1-indexed 数组
 * @param {number} target - 目标和
 * @return {number[]} - 返回 1-indexed 的索引数组
 */
var twoSum = function (numbers, target) {
  let left = 0;
  let right = numbers.length - 1;

  while (left < right) {
    const sum = numbers[left] + numbers[right];

    if (sum === target) {
      // 题目要求返回 1-indexed，所以索引需要加 1
      return [left + 1, right + 1];
    } else if (sum < target) {
      // 如果和太小，说明需要更大的数，左指针右移
      left++;
    } else {
      // 如果和太大，说明需要更小的数，右指针左移
      right--;
    }
  }

  // 题目保证有解，正常情况下不会执行到这里
  return [];
};
```

### Container With Most Water: 贪心证明与双指针结合。

```javascript
/**
 * @param {number[]} height
 * @return {number}
 */
var maxArea = function (height) {
  let maxArea = 0;
  let left = 0;
  let right = height.length - 1;

  while (left < right) {
    // 计算当前窗口的宽度
    const width = right - left;
    // 容器的高度由较短的那根柱子决定
    const currentHeight = Math.min(height[left], height[right]);

    // 更新最大面积
    const area = width * currentHeight;
    maxArea = Math.max(maxArea, area);

    // 核心贪心策略：移动较短的那根柱子
    // 因为容器的高度受限于短板，如果移动长板，宽度变小且高度依然受限于原短板，面积只会减小。
    // 只有移动短板，才有可能遇到更高柱子从而抵消宽度减小的损失。
    if (height[left] < height[right]) {
      left++;
    } else {
      right--;
    }
  }

  return maxArea;
};
```

### 3Sum: 排序+双指针，去重逻辑是面试重点。

解决 3Sum 的核心思路是：先对数组进行排序，然后固定一个数 nums[i]，再利用双指针在剩余的数组部分寻找另外两个数，使得三数之和为零。

```javascript
/**
 * @param {number[]} nums
 * @return {number[][]}
 */
var threeSum = function (nums) {
  const res = [];
  // 1. 必须先排序，方便去重和使用双指针
  nums.sort((a, b) => a - b);

  for (let i = 0; i < nums.length - 2; i++) {
    // 如果当前数已经大于 0，由于数组有序，后面的数之和必然大于 0
    if (nums[i] > 0) break;

    // 2. 去重：如果当前数和前一个数相同，跳过以避免重复三元组
    if (i > 0 && nums[i] === nums[i - 1]) continue;

    let left = i + 1;
    let right = nums.length - 1;

    while (left < right) {
      const sum = nums[i] + nums[left] + nums[right];

      if (sum === 0) {
        res.push([nums[i], nums[left], nums[right]]);

        // 3. 找到答案后，继续移动指针并跳过重复值
        while (left < right && nums[left] === nums[left + 1]) left++;
        while (left < right && nums[right] === nums[right - 1]) right--;

        left++;
        right--;
      } else if (sum < 0) {
        left++; // 和太小，左指针右移
      } else {
        right--; // 和太大，右指针左移
      }
    }
  }

  return res;
};
```

## 2. 滑动窗口与哈希表 (6 题)

滑动窗口是处理子串、子数组问题的万能钥匙。

### Minimum Size Subarray Sum: 动态窗口大小的经典题。

由于数组中全是正整数，我们可以确定：窗口越宽，和越大；窗口越窄，和越小。这种单调性让我们能够通过调整左右边界，在线性时间内找到最优解。

```javascript
/**
 * @param {number} target
 * @param {number[]} nums
 * @return {number}
 */
var minSubArrayLen = function (target, nums) {
  let left = 0;
  let sum = 0;
  let minLength = Infinity;

  // right 指针不断向右移动，扩张窗口
  for (let right = 0; right < nums.length; right++) {
    sum += nums[right];

    // 一旦当前窗口的和满足条件，尝试通过收缩左边界来减小窗口长度
    while (sum >= target) {
      // 更新最小长度
      minLength = Math.min(minLength, right - left + 1);

      // 减去左侧元素，并将左指针右移
      sum -= nums[left];
      left++;
    }
  }

  // 如果 minLength 没被更新过，说明不存在符合条件的子数组
  return minLength === Infinity ? 0 : minLength;
};
```

### Longest Substring Without Repeating Characters: 哈希表配合窗口滑动的标准模版。

使用滑动窗口结合哈希表（Map）。我们利用窗口 $[left, right]$ 扫描字符串，哈希表用于记录每个字符最后出现的位置。当遇到重复字符时，直接将左边界 left 跳过重复点，从而保持窗口内字符的唯一性。

```javascript
/**
 * @param {string} s
 * @return {number}
 */
var lengthOfLongestSubstring = function (s) {
  // map 存储字符及其最后出现的索引 {字符: 索引}
  const charMap = new Map();
  let maxLength = 0;
  let left = 0; // 窗口左边界

  for (let right = 0; right < s.length; right++) {
    const char = s[right];

    // 如果字符已存在于当前窗口内
    if (charMap.has(char)) {
      // 核心逻辑：将左边界跳到重复字符上次出现位置的下一个点
      // 注意：必须取 Math.max，确保 left 不会往回跳（处理不在当前窗口内的历史记录）
      left = Math.max(left, charMap.get(char) + 1);
    }

    // 更新字符最新的索引位置
    charMap.set(char, right);

    // 计算当前窗口长度并更新最大值
    maxLength = Math.max(maxLength, right - left + 1);
  }

  return maxLength;
};
```

### Minimum Window Substring: (高频 Hard) 滑动窗口最复杂的形态。

我们需要在 $s$ 中寻找一个最短的窗口，使其包含 $t$ 中所有的字符及其出现的频率。

```javascript
/**
 * @param {string} s
 * @param {string} t
 * @return {string}
 */
var minWindow = function (s, t) {
  if (s.length < t.length) return "";

  // 1. 统计 t 中字符出现的频率
  const need = {};
  for (let char of t) {
    need[char] = (need[char] || 0) + 1;
  }

  // window 记录当前窗口内字符的频率
  const window = {};
  let left = 0;
  let right = 0;
  // valid 表示窗口内已经满足 t 中字符频率条件的字符个数
  let valid = 0;

  // 记录最小覆盖子串的起始索引及长度
  let start = 0;
  let minLen = Infinity;

  const requiredCount = Object.keys(need).length;

  while (right < s.length) {
    // a 是将移入窗口的字符
    let a = s[right];
    right++;

    // 进行窗口内数据的系列更新
    if (need[a]) {
      window[a] = (window[a] || 0) + 1;
      if (window[a] === need[a]) {
        valid++;
      }
    }

    // 判断左侧窗口是否收缩
    while (valid === requiredCount) {
      // 在这里更新最小覆盖子串
      if (right - left < minLen) {
        start = left;
        minLen = right - left;
      }

      // d 是将移出窗口的字符
      let d = s[left];
      left++;

      // 进行窗口内数据的系列更新
      if (need[d]) {
        if (window[d] === need[d]) {
          valid--;
        }
        window[d]--;
      }
    }
  }

  return minLen === Infinity ? "" : s.substr(start, minLen);
};
```

### Group Anagrams: 哈希表处理字符串分类。

所谓“字母异位词”（Anagram），是指两个字符串包含的字母种类和数量完全相同，只是顺序不同。

要将它们分组，我们需要找到一个通用的“键” (Key)，使得所有互为字母异位词的字符串都能映射到同一个键上。

如果字符串非常长，排序会变成瓶颈 ($O(L \log L)$)。我们可以利用题目中“仅包含小写字母”的约束，用长度为 26 的频次数组作为键。

```javascript
/**
 * 这种方法在处理大量长字符串时效率更高
 */
var groupAnagrams = function (strs) {
  const map = {};

  for (let str of strs) {
    // 创建一个长度为 26 的计数器
    const count = new Array(26).fill(0);
    for (let char of str) {
      count[char.charCodeAt(0) - "a".charCodeAt(0)]++;
    }

    // 将数组转换为字符串作为 key，例如 "1,0,1,0..."
    const key = count.join(",");

    if (!map[key]) map[key] = [];
    map[key].push(str);
  }

  return Object.values(map);
};
```

### Longest Consecutive Sequence: 利用哈希表将 $O(n \log n)$ 优化至 $O(n)$。

我们必须利用 哈希表（Set） 来实现 $O(1)$ 级别的查询。核心逻辑是：对于数组中的每个数，我们只在它是序列的起点时，才开始向后计数。

```javascript
/**
 * @param {number[]} nums
 * @return {number}
 */
var longestConsecutive = function (nums) {
  // 1. 将所有数字存入 Set，去除重复且实现 O(1) 查询
  const numSet = new Set(nums);
  let longestStreak = 0;

  for (const num of numSet) {
    // 2. 关键点：判断当前数字是否是一个序列的“起点”
    // 如果 num - 1 存在于 Set 中，说明 num 只是序列中间的一个数
    // 我们跳过它，等到遍历到真正的起点时再处理
    if (!numSet.has(num - 1)) {
      let currentNum = num;
      let currentStreak = 1;

      // 3. 从起点开始，不断查找下一个连续的数字
      while (numSet.has(currentNum + 1)) {
        currentNum += 1;
        currentStreak += 1;
      }

      // 4. 更新全局最长长度
      longestStreak = Math.max(longestStreak, currentStreak);
    }
  }

  return longestStreak;
};
```

### Two Sum: 面试第一题，考查对哈希表的理解。

```javascript
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
var twoSum = function (nums, target) {
  const map = new Map();

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (map.has(complement)) {
      return [map.get(complement), i];
    }
    map.set(nums[i], i);
  }
};
```

## 3. 矩阵与区间 (5 题)

考查坐标变换和模拟能力。

### Spiral Matrix: 经典的螺旋遍历模拟。

解决螺旋遍历最稳妥的方法是定义四个边界：top（上）、bottom（下）、left（左）、right（右）。每当我们遍历完一个方向，就收缩对应的边界。

```javascript
/**
 * @param {number[][]} matrix
 * @return {number[]}
 */
var spiralOrder = function (matrix) {
  if (matrix.length === 0) return [];

  const result = [];
  // 1. 初始化四个边界
  let top = 0;
  let bottom = matrix.length - 1;
  let left = 0;
  let right = matrix[0].length - 1;

  // 当边界没有交叠时，继续循环
  while (left <= right && top <= bottom) {
    // 方向一：从左往右
    for (let i = left; i <= right; i++) {
      result.push(matrix[top][i]);
    }
    top++; // 遍历完一行，上边界下移

    // 方向二：从上往下
    for (let i = top; i <= bottom; i++) {
      result.push(matrix[i][right]);
    }
    right--; // 遍历完一列，右边界左移

    // 关键：在向左和向上遍历前，必须再次检查边界
    // 否则在非对称矩阵（如只有一行或一列）中会重复遍历
    if (top <= bottom) {
      // 方向三：从右往左
      for (let i = right; i >= left; i--) {
        result.push(matrix[bottom][i]);
      }
      bottom--; // 下边界上移
    }

    if (left <= right) {
      // 方向四：从下往上
      for (let i = bottom; i >= top; i--) {
        result.push(matrix[i][left]);
      }
      left++; // 左边界右移
    }
  }

  return result;
};
```

### Rotate Image: 矩阵翻转代替旋转的技巧。

最优雅的解法是利用线性代数的几何特性，将顺时针旋转 $90^{\circ}$ 拆解为两个简单的步骤：转置（Transpose）：将矩阵的行变为列（沿主对角线翻转）。镜像（Reflect）：将每一行进行左右反转。

```javascript
/**
 * @param {number[][]} matrix
 * @return {void} Do not return anything, modify matrix in-place instead.
 */
var rotate = function (matrix) {
  const n = matrix.length;

  // 1. 转置矩阵 (沿主对角线对折)
  // 使得 matrix[i][j] 与 matrix[j][i] 交换
  for (let i = 0; i < n; i++) {
    // 注意 j 从 i 开始，避免重复交换导致换回原样
    for (let j = i; j < n; j++) {
      [matrix[i][j], matrix[j][i]] = [matrix[j][i], matrix[i][j]];
    }
  }

  // 2. 镜像反转 (每一行进行左右翻转)
  for (let i = 0; i < n; i++) {
    // 直接使用数组的 reverse 方法或双指针手动交换
    matrix[i].reverse();
  }
};
```

### Set Matrix Zeroes: 巧妙利用原矩阵第一行/列做标记位。

这道题的难点在于**原地（In-place）**操作。如果我们直接在遍历时发现 0 就把行和列置为 0，那么后续的遍历就会被这些新生成的 0 干扰，导致整个矩阵最终都变成 0。常规解法需要 $O(m + n)$ 的额外空间来记录哪些行和列需要置零，但为了达到面试进阶要求的 $O(1)$ 空间，我们可以利用矩阵的第一行和第一列来充当“标记位”。

```javascript
/**
 * @param {number[][]} matrix
 * @return {void} Do not return anything, modify matrix in-place instead.
 */
var setZeroes = function (matrix) {
  const m = matrix.length;
  const n = matrix[0].length;

  // 因为第一行和第一列会被用作标记位，所以需要先记录它们本身是否包含 0
  let firstRowHasZero = false;
  let firstColHasZero = false;

  // 1. 检查第一列
  for (let i = 0; i < m; i++) {
    if (matrix[i][0] === 0) firstColHasZero = true;
  }

  // 2. 检查第一行
  for (let j = 0; j < n; j++) {
    if (matrix[0][j] === 0) firstRowHasZero = true;
  }

  // 3. 使用第一行和第一列记录内部区域（1,1 到 m,n）的零情况
  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) {
      if (matrix[i][j] === 0) {
        matrix[i][0] = 0; // 标记该行需要置零
        matrix[0][j] = 0; // 标记该列需要置零
      }
    }
  }

  // 4. 根据第一行和第一列的标记，对内部区域进行置零
  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) {
      if (matrix[i][0] === 0 || matrix[0][j] === 0) {
        matrix[i][j] = 0;
      }
    }
  }

  // 5. 最后单独处理第一行和第一列
  if (firstColHasZero) {
    for (let i = 0; i < m; i++) matrix[i][0] = 0;
  }
  if (firstRowHasZero) {
    for (let j = 0; j < n; j++) matrix[0][j] = 0;
  }
};
```

### Merge Intervals: 区间问题的核心：排序 + 合并逻辑。

如果不排序，区间可能以任意顺序出现，你就必须不断回溯对比，复杂度会失控。一旦按起始位置排序，我们只需要关注当前区间是否与上一个“合并后的区间”产生交集。

```javascript
/**
 * @param {number[][]} intervals
 * @return {number[][]}
 */
var merge = function (intervals) {
  if (intervals.length <= 1) return intervals;

  // 1. 核心步骤：按区间的起始位置进行升序排序
  intervals.sort((a, b) => a[0] - b[0]);

  const result = [];
  // 初始化第一个区间
  let currInterval = intervals[0];
  result.push(currInterval);

  for (let i = 1; i < intervals.length; i++) {
    let nextInterval = intervals[i];

    // 2. 逻辑判断：如果当前区间的终点 >= 下一个区间的起点，说明重叠了
    if (currInterval[1] >= nextInterval[0]) {
      // 合并区间：将当前区间的终点更新为两者终点的最大值
      currInterval[1] = Math.max(currInterval[1], nextInterval[1]);
    } else {
      // 3. 如果不重叠，则将下一个区间作为“当前待合并区间”存入结果
      currInterval = nextInterval;
      result.push(currInterval);
    }
  }

  return result;
};
```

### Insert Interval: 处理区间重叠的细致考查。

既然输入的 intervals 已经是有序且不重叠的，我们不需要重新排序，而是可以利用这一特性，通过一次遍历将区间分为三个阶段处理：

左侧无重叠阶段：当前区间的终点小于新区间的起点。

重叠合并阶段：当前区间与新区间有交集（只要当前区间的起点小于等于新区间的终点，且终点大于等于新区间的起点）。

右侧无重叠阶段：当前区间的起点大于新区间的终点。

```javascript
/**
 * @param {number[][]} intervals
 * @param {number[]} newInterval
 * @return {number[][]}
 */
var insert = function (intervals, newInterval) {
  const result = [];
  let i = 0;
  const n = intervals.length;

  // 1. 处理所有在新区间左侧且无重叠的区间
  while (i < n && intervals[i][1] < newInterval[0]) {
    result.push(intervals[i]);
    i++;
  }

  // 2. 处理所有与新区间有重叠的区间，并不断更新新区间（合并）
  // 只要当前区间的起点 <= 新区间的终点，就说明还有重叠可能
  while (i < n && intervals[i][0] <= newInterval[1]) {
    newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
    newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
    i++;
  }
  // 将合并后的巨大“新区间”放入结果
  result.push(newInterval);

  // 3. 处理所有在新区间右侧且无重叠的区间
  while (i < n) {
    result.push(intervals[i]);
    i++;
  }

  return result;
};
```

## 4. 栈与链表 (7 题)

链表重点在于指针操作，栈重点在于匹配和模拟。

- Valid Parentheses: 栈的入门必修课。
  由于括号必须以“后进先出”的顺序匹配（即最后打开的左括号必须最先被闭合），栈成为了处理此类问题的完美工具。

```javascript
/**
 * @param {string} s
 * @return {boolean}
 */
var isValid = function (s) {
  // 1. 建立一个映射表，方便通过右括号查找对应的左括号
  const map = {
    ")": "(",
    "}": "{",
    "]": "[",
  };

  const stack = [];

  for (let char of s) {
    // 2. 如果是右括号
    if (map[char]) {
      // 弹出栈顶元素进行匹配
      // 如果栈为空（说明右括号多余）或者匹配失败，直接返回 false
      const topElement = stack.length === 0 ? "#" : stack.pop();
      if (topElement !== map[char]) {
        return false;
      }
    } else {
      // 3. 如果是左括号，直接入栈
      stack.push(char);
    }
  }

  // 4. 最后检查栈是否为空。如果为空，说明全部匹配成功；否则说明左括号多余。
  return stack.length === 0;
};
```

### Min Stack: 辅助栈的设计思想。

为了实现常数级查找，我们需要“空间换时间”。

```javascript
var MinStack = function () {
  // 主栈：存储所有元素
  this.stack = [];
  // 辅助栈：存储到当前为止的最小值
  this.minStack = [];
};

/** * @param {number} val
 * @return {void}
 */
MinStack.prototype.push = function (val) {
  this.stack.push(val);

  // 如果辅助栈为空，或者当前值小于等于辅助栈顶（当前最小值），则入辅助栈
  if (
    this.minStack.length === 0 ||
    val <= this.minStack[this.minStack.length - 1]
  ) {
    this.minStack.push(val);
  }
};

/**
 * @return {void}
 */
MinStack.prototype.pop = function () {
  const poppedValue = this.stack.pop();

  // 如果主栈弹出的值正是当前的最小值，辅助栈也要同步弹出
  if (poppedValue === this.minStack[this.minStack.length - 1]) {
    this.minStack.pop();
  }
};

/**
 * @return {number}
 */
MinStack.prototype.top = function () {
  return this.stack[this.stack.length - 1];
};

/**
 * @return {number}
 */
MinStack.prototype.getMin = function () {
  return this.minStack[this.minStack.length - 1];
};
```

### Linked List Cycle: 快慢指针（判环）的起源。

这是面试官最想听到的解法。我们设置两个指针：慢指针 (slow) 每次走一步，快指针 (fast) 每次走两步。

如果链表中没有环，快指针最终会到达终点（null）。

如果链表中有环，快指针最终会进入环并在环内绕圈。由于快慢指针的速度差为 1，快指针一定会从后方“追上”慢指针。

```javascript
/**
 * @param {ListNode} head
 * @return {boolean}
 */
var hasCycle = function (head) {
  if (!head || !head.next) return false;

  let slow = head;
  let fast = head;

  // 只要 fast 及其下一步不为空，就继续移动
  while (fast && fast.next) {
    slow = slow.next; // 走一步
    fast = fast.next.next; // 走两步

    // 如果相遇，说明有环
    if (slow === fast) {
      return true;
    }
  }

  // fast 到达末尾，说明无环
  return false;
};
```

### Add Two Numbers: 链表模拟大数加法。

我们可以直接从头节点开始相加，处理进位逻辑，这正符合我们手工加法的习惯。

```javascript
/**
 * Definition for singly-linked list.
 * function ListNode(val, next) {
 * this.val = (val===undefined ? 0 : val)
 * this.next = (next===undefined ? null : next)
 * }
 */
/**
 * @param {ListNode} l1
 * @param {ListNode} l2
 * @return {ListNode}
 */
var addTwoNumbers = function (l1, l2) {
  // 1. 创建一个哑结点 (Dummy Node) 作为结果链表的起点
  const dummy = new ListNode(0);
  let curr = dummy;
  let carry = 0; // 进位

  // 2. 只要 l1, l2 没遍历完，或者还有进位，就继续计算
  while (l1 !== null || l2 !== null || carry !== 0) {
    // 获取当前节点的值，如果链表已经到头，则取 0
    const x = l1 ? l1.val : 0;
    const y = l2 ? l2.val : 0;

    // 计算总和及新的进位
    const sum = x + y + carry;
    carry = Math.floor(sum / 10);

    // 将个位数存入新节点
    curr.next = new ListNode(sum % 10);
    curr = curr.next;

    // 移动指针
    if (l1) l1 = l1.next;
    if (l2) l2 = l2.next;
  }

  // 3. 返回哑结点的下一个节点，即真正的结果头节点
  return dummy.next;
};
```

### Reverse Linked List II: 局部反转链表，考察指针断开与重连的细节。

它的难点在于：你不仅要反转中间那段链表，还要把反转后的部分与原链表的**前驱（Predecessor）和后继（Successor）**重新缝合起来。

```javascript
/**
 * @param {ListNode} head
 * @param {number} left
 * @param {number} right
 * @return {ListNode}
 */
var reverseBetween = function (head, left, right) {
  // 1. 使用哑结点处理 left = 1 的特殊情况
  const dummy = new ListNode(0);
  dummy.next = head;

  // 2. 找到待反转部分的前驱节点 prev
  let prev = dummy;
  for (let i = 0; i < left - 1; i++) {
    prev = prev.next;
  }

  // 3. 开始局部反转
  // start 是反转部分的第一个节点，反转后它会变成这部分的最后一个
  // then 是 start 的下一个节点，我们将不断把 then 移到 prev 的后面
  let start = prev.next;
  let then = start.next;

  // 需要执行 right - left 次跨越式交换
  for (let i = 0; i < right - left; i++) {
    start.next = then.next;
    then.next = prev.next;
    prev.next = then;
    then = start.next;
  }

  return dummy.next;
};
```

### Copy List with Random Pointer: 哈希表或原地拆分节点的进阶操作。

通过在原节点后面直接插入克隆节点，我们可以利用这种物理位置的邻近性来寻找 random 节点，从而将空间复杂度降到 $O(1)$。

```javascript
var copyRandomList = function (head) {
  if (!head) return null;

  // 1. 在每个原节点后面插入克隆节点
  // A -> B  变为  A -> A' -> B -> B'
  let curr = head;
  while (curr) {
    let next = curr.next;
    curr.next = new Node(curr.val, next);
    curr = next;
  }

  // 2. 设置克隆节点的 random 指针
  // A'.random = A.random.next (即 A.random 的克隆版)
  curr = head;
  while (curr) {
    if (curr.random) {
      curr.next.random = curr.random.next;
    }
    curr = curr.next.next;
  }

  // 3. 拆分两个链表，恢复原链表并提取克隆链表
  curr = head;
  let dummy = new Node(0);
  let copyCurr = dummy;
  while (curr) {
    copyCurr.next = curr.next;
    copyCurr = copyCurr.next;

    curr.next = curr.next.next; // 恢复原链表
    curr = curr.next;
  }

  return dummy.next;
};
```

### LRU Cache: (必考 Medium) 哈希表 + 双向链表的工程实践。

最近使用的在前：每次 get 或 put 现有的 key 时，将对应的节点移动到链表头部。

最久未使用的在后：当缓存满时，直接移除链表尾部的节点，并同步删除哈希表中的记录。

哨兵节点 (Dummy Nodes)：使用 head 和 tail 两个伪节点，可以极大简化边界处理（如处理空链表）。

```javascript
class Node {
  constructor(key, value) {
    this.key = key;
    this.value = value;
    this.prev = null;
    this.next = null;
  }
}

/**
 * @param {number} capacity
 */
var LRUCache = function (capacity) {
  this.capacity = capacity;
  this.map = new Map(); // key -> Node

  // 初始化双向链表哨兵
  this.head = new Node(0, 0);
  this.tail = new Node(0, 0);
  this.head.next = this.tail;
  this.tail.prev = this.head;
};

/** * @param {number} key
 * @return {number}
 */
LRUCache.prototype.get = function (key) {
  if (this.map.has(key)) {
    const node = this.map.get(key);
    this._moveToHead(node); // 访问后标记为“最近使用”
    return node.value;
  }
  return -1;
};

/** * @param {number} key
 * @param {number} value
 * @return {void}
 */
LRUCache.prototype.put = function (key, value) {
  if (this.map.has(key)) {
    const node = this.map.get(key);
    node.value = value;
    this._moveToHead(node);
  } else {
    const newNode = new Node(key, value);
    this.map.set(key, newNode);
    this._addNode(newNode);

    if (this.map.size > this.capacity) {
      const lru = this._popTail();
      this.map.delete(lru.key);
    }
  }
};

// --- 内部辅助方法 ---

LRUCache.prototype._addNode = function (node) {
  // 始终插入到 head 的后面
  node.prev = this.head;
  node.next = this.head.next;
  this.head.next.prev = node;
  this.head.next = node;
};

LRUCache.prototype._removeNode = function (node) {
  // 将节点从链表中摘除
  const prev = node.prev;
  const next = node.next;
  prev.next = next;
  next.prev = prev;
};

LRUCache.prototype._moveToHead = function (node) {
  this._removeNode(node);
  this._addNode(node);
};

LRUCache.prototype._popTail = function () {
  // 弹出 tail 前面的那个真实节点
  const res = this.tail.prev;
  this._removeNode(res);
  return res;
};
```

## 5. 二叉树 (8 题)

递归是二叉树的灵魂，必须熟练掌握 DFS 和 BFS。

### Invert Binary Tree: 递归思想的最直观体现。

递归地交换每一个节点的左右子节点。

```javascript
/**
 * @param {TreeNode} root
 * @return {TreeNode}
 */
var invertTree = function (root) {
  // 1. 终止条件：如果节点为空，直接返回
  if (root === null) return null;

  // 2. 暂时保存翻转后的左子树和右子树
  const left = invertTree(root.left);
  const right = invertTree(root.right);

  // 3. 交换当前节点的左右子树
  root.left = right;
  root.right = left;

  // 4. 返回当前节点
  return root;
};
```

### Construct Binary Tree from Preorder and Inorder Traversal: 递归构建树的核心逻辑。

我们通过递归来不断构建子树：

从 preorder 取出第一个元素作为根节点。

在 inorder 中找到该根节点的位置（索引 i）。

计算左子树的节点数量：i - inorderStart。

根据这个数量，将 preorder 和 inorder 划分为左、右两部分，递归构建。

为了提高查找 inorder 索引的效率，我们通常先用一个 哈希表 存储中序遍历的值与索引的映射。

```javascript
/**
 * @param {number[]} preorder
 * @param {number[]} inorder
 * @return {TreeNode}
 */
var buildTree = function (preorder, inorder) {
  // 1. 哈希表缓存中序遍历的索引，优化查找效率到 O(1)
  const map = new Map();
  for (let i = 0; i < inorder.length; i++) {
    map.set(inorder[i], i);
  }

  const build = (preStart, preEnd, inStart, inEnd) => {
    // 递归终止条件
    if (preStart > preEnd || inStart > inEnd) return null;

    // 2. 确定根节点：前序遍历的第一个元素
    const rootVal = preorder[preStart];
    const root = new TreeNode(rootVal);

    // 3. 在中序遍历中定位根节点
    const mid = map.get(rootVal);

    // 4. 计算左子树节点个数
    const leftSize = mid - inStart;

    // 5. 递归构建
    // 左子树在前序中的范围：[preStart + 1, preStart + leftSize]
    root.left = build(preStart + 1, preStart + leftSize, inStart, mid - 1);

    // 右子树在前序中的范围：[preStart + leftSize + 1, preEnd]
    root.right = build(preStart + leftSize + 1, preEnd, mid + 1, inEnd);

    return root;
  };

  return build(0, preorder.length - 1, 0, inorder.length - 1);
};
```

### Lowest Common Ancestor of a Binary Tree: 递归寻找公共祖先，逻辑精妙。

对于任意一个节点，我们向上汇报三种情况：

如果当前节点是 p 或 q，直接返回当前节点。

如果当前节点不是目标，去左右子树找：

如果左右子树各返回了一个目标（一个找到了 p，一个找到了 q），那么当前节点就是 LCA。

如果只有一侧返回了非空值，说明 p 和 q 都在那一侧，继续向上返回该非空值。

如果两侧都返回空，说明这一支路没有目标。

```javascript
/**
 * @param {TreeNode} root
 * @param {TreeNode} p
 * @param {TreeNode} q
 * @return {TreeNode}
 */
var lowestCommonAncestor = function (root, p, q) {
  // 1. 终止条件：如果遇到了空节点，或者遇到了 p 或 q，直接返回
  if (root === null || root === p || root === q) {
    return root;
  }

  // 2. 递归去左子树和右子树寻找 p 和 q
  const left = lowestCommonAncestor(root.left, p, q);
  const right = lowestCommonAncestor(root.right, p, q);

  // 3. 逻辑判断（根据左右子树的返回结果）：

  // 情况 A：左、右子树都找到了目标（一边一个）
  // 那么当前的 root 就是离它们最近的公共祖先
  if (left !== null && right !== null) {
    return root;
  }

  // 情况 B：只有左子树找到了（p 和 q 都在左边，或者只找到了其中一个）
  if (left !== null) {
    return left;
  }

  // 情况 C：只有右子树找到了
  if (right !== null) {
    return right;
  }

  // 情况 D：左右都没找到
  return null;
};
```

### Binary Tree Level Order Traversal: BFS 的标准层序遍历。

为了实现这一点，我们使用 队列 (Queue) 来辅助遍历。

```javascript
/**
 * @param {TreeNode} root
 * @return {number[][]}
 */
var levelOrder = function (root) {
  if (!root) return [];

  const result = [];
  const queue = [root]; // 1. 初始化队列，将根节点入队

  while (queue.length > 0) {
    // 2. 记录当前层的节点个数
    // 这一步是关键：它保证了我们每次只处理当前层的所有节点
    const levelSize = queue.length;
    const currentLevel = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift(); // 3. 弹出队首节点
      currentLevel.push(node.val);

      // 4. 将左右子节点（下一层）加入队列
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    // 5. 将当前层的结果存入最终结果数组
    result.push(currentLevel);
  }

  return result;
};
```

### Binary Tree Zigzag Level Order Traversal: 层序遍历的变形。

```javascript
/**
 * @param {TreeNode} root
 * @return {number[][]}
 */
var zigzagLevelOrder = function (root) {
  if (!root) return [];

  const result = [];
  const queue = [root];
  let leftToRight = true; // 方向控制标志位

  while (queue.length > 0) {
    const levelSize = queue.length;
    const currentLevel = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();

      // 根据当前方向，决定是将值“推入”末尾还是“插入”头部
      if (leftToRight) {
        currentLevel.push(node.val);
      } else {
        currentLevel.unshift(node.val);
      }

      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    result.push(currentLevel);
    leftToRight = !leftToRight; // 每一层处理完后，切换方向
  }

  return result;
};
```

### Binary Tree Maximum Path Sum: (高频 Hard) 树形 DP 的经典。

一个节点在路径中可以扮演两种角色：

路径的“拐点”：路径从左子树上来，经过该节点，再下到右子树。这种情况下，该节点及其左右子树路径构成了完整的路径，不能再往父节点延伸。

路径的“分支”：路径从该节点向上延伸到父节点。这种情况下，该节点只能选择左子树或右子树中贡献较大的那一侧。

```javascript
/**
 * @param {TreeNode} root
 * @return {number}
 */
var maxPathSum = function (root) {
  let maxSum = -Infinity;

  const gainFromNode = (node) => {
    if (!node) return 0;

    // 1. 递归计算左右子树能提供的最大贡献
    // 如果子树贡献为负，我们宁愿不选（取 0）
    const leftGain = Math.max(gainFromNode(node.left), 0);
    const rightGain = Math.max(gainFromNode(node.right), 0);

    // 2. 计算以当前节点为“拐点”的路径和
    // 这条路径连接了左子树、当前节点、右子树
    const currentPathSum = node.val + leftGain + rightGain;

    // 3. 更新全局最大值
    maxSum = Math.max(maxSum, currentPathSum);

    // 4. 返回该节点给父节点提供的最大贡献
    // 只能选择左或者右的一条路径，加上节点自身值
    return node.val + Math.max(leftGain, rightGain);
  };

  gainFromNode(root);
  return maxSum;
};
```

### Validate Binary Search Tree: BST 性质的深度考查。

正如我们在处理 Kth Smallest Element 时提到的，BST 的中序遍历结果必然是严格递增的。我们可以利用这个性质进行校验。

```javascript
var isValidBST = function (root) {
  let prev = -Infinity;
  let isValid = true;

  const inOrder = (node) => {
    if (!node || !isValid) return;

    inOrder(node.left);

    // 如果当前值不大于前一个值，说明不是合法的 BST
    if (node.val <= prev) {
      isValid = false;
      return;
    }
    prev = node.val;

    inOrder(node.right);
  };

  inOrder(root);
  return isValid;
};
```

### Kth Smallest Element in a BST: 利用 BST 的中序遍历有序性。

使用栈进行中序遍历。这种方法的好处是可以在找到第 $k$ 个元素时立即停止，而不需要遍历整棵树，效率更高。

```javascript
/**
 * @param {TreeNode} root
 * @param {number} k
 * @return {number}
 */
var kthSmallest = function (root, k) {
  const stack = [];
  let curr = root;

  // 经典的中序遍历迭代模版
  while (curr || stack.length > 0) {
    // 1. 尽可能向左走，把路上的节点都压入栈
    while (curr) {
      stack.push(curr);
      curr = curr.left;
    }

    // 2. 弹出栈顶节点（当前最左/最小的节点）
    curr = stack.pop();

    // 3. 计数，如果达到 k，则找到了目标
    k--;
    if (k === 0) return curr.val;

    // 4. 转向右子树
    curr = curr.right;
  }
};
```

## 6. 图与回溯 (6 题)

图论考查搜索算法，回溯考查对状态树的搜索与剪枝。

### Number of Islands: 网格类 DFS/BFS 的母题。

```javascript
/**
 * @param {character[][]} grid
 * @return {number}
 */
var numIslands = function (grid) {
  if (!grid || grid.length === 0) return 0;

  let islandCount = 0;
  const rows = grid.length;
  const cols = grid[0].length;

  // DFS 辅助函数：用于淹没连通的陆地
  const dfs = (r, c) => {
    // 边界检查：越界或遇到水 ('0') 则停止
    if (r < 0 || c < 0 || r >= rows || c >= cols || grid[r][c] === "0") {
      return;
    }

    // 将当前陆地标记为 '0'，防止重复访问
    grid[r][c] = "0";

    // 向四个方向递归探索
    dfs(r + 1, c); // 下
    dfs(r - 1, c); // 上
    dfs(r, c + 1); // 右
    dfs(r, c - 1); // 左
  };

  // 遍历整个网格
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      // 每当我们发现一个 '1'，说明发现了一个新的岛屿
      if (grid[r][c] === "1") {
        islandCount++;
        // 启动 DFS，把属于这个岛屿的所有陆地全部变成 '0'
        dfs(r, c);
      }
    }
  }

  return islandCount;
};
```

### Course Schedule: 拓扑排序（判环）的经典应用。

我们通常使用 Kahn 算法 (基于 BFS) 来解决，它通过维护每个节点的“入度”（in-degree，即指向该节点的边数）来工作。

```javascript
/**
 * @param {number} numCourses
 * @param {number[][]} prerequisites
 * @return {boolean}
 */
var canFinish = function (numCourses, prerequisites) {
  // 1. 构建邻接表和入度数组
  const inDegree = new Array(numCourses).fill(0);
  const adj = Array.from({ length: numCourses }, () => []);

  for (let [course, pre] of prerequisites) {
    adj[pre].push(course); // pre -> course
    inDegree[course]++; // 课程 course 的先修课多了一个
  }

  // 2. 将所有入度为 0 的课程（没有先修课）放入队列
  const queue = [];
  for (let i = 0; i < numCourses; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  // 3. 开始 BFS
  let count = 0; // 记录修过的课程数量
  while (queue.length > 0) {
    const curr = queue.shift();
    count++;

    // 遍历当前课程的所有后续课程
    for (let nextCourse of adj[curr]) {
      inDegree[nextCourse]--; // 既然先修课 curr 修完了，后续课程入度减 1

      // 如果后续课程的入度变为 0，说明它也可以修了
      if (inDegree[nextCourse] === 0) {
        queue.push(nextCourse);
      }
    }
  }

  // 4. 如果修过的课程数量等于总课程数，说明没有环
  return count === numCourses;
};
```

### Clone Graph: 图的深拷贝（DFS/BFS）。

对于图这种具有环状结构的数据，最关键的挑战是避免死循环。我们需要一个哈希表 (Map) 来记录已经克隆过的节点。如果遇到已经克隆过的节点，直接返回克隆后的引用，而不是再次克隆。

```javascript
/**
 * @param {Node} node
 * @return {Node}
 */
var cloneGraph = function (node) {
  if (!node) return null;

  // 存储已经克隆过的节点：key 为原节点，value 为克隆节点
  const visited = new Map();

  const dfs = (currNode) => {
    // 如果该节点已经克隆过，直接返回克隆后的引用
    if (visited.has(currNode)) {
      return visited.get(currNode);
    }

    // 1. 克隆当前节点（只克隆值，不克隆邻居）
    const cloneNode = new Node(currNode.val);
    visited.set(currNode, cloneNode);

    // 2. 递归克隆邻居，并建立连接
    for (let neighbor of currNode.neighbors) {
      cloneNode.neighbors.push(dfs(neighbor));
    }

    return cloneNode;
  };

  return dfs(node);
};
```

### Letter Combinations of a Phone Number: 回溯算法处理组合。

你可以将这个问题想象成一棵决策树：每一位数字代表树的一层，而该数字对应的每一个字母则是你从当前节点出发的分支。

```javascript
/**
 * @param {string} digits
 * @return {string[]}
 */
var letterCombinations = function (digits) {
  if (!digits.length) return [];

  // 1. 建立数字到字母的映射表
  const phoneMap = {
    2: "abc",
    3: "def",
    4: "ghi",
    5: "jkl",
    6: "mno",
    7: "pqrs",
    8: "tuv",
    9: "wxyz",
  };

  const result = [];

  /**
   * @param {number} index 当前处理到 digits 的第几个数字
   * @param {string} path 当前已经构建出来的字符串片段
   */
  const backtrack = (index, path) => {
    // 终止条件：如果路径长度等于输入数字长度，说明找全了一个组合
    if (path.length === digits.length) {
      result.push(path);
      return;
    }

    // 获取当前数字对应的所有字母
    const letters = phoneMap[digits[index]];

    // 遍历所有可能的字母分支
    for (const char of letters) {
      // 递归进入下一层（处理下一个数字）
      backtrack(index + 1, path + char);
    }
  };

  backtrack(0, "");
  return result;
};
```

### Permutations: 回溯算法处理全排列。

为了实现这一点，我们需要在回溯过程中维护一个“已使用”的状态。

```javascript
/**
 * @param {number[]} nums
 * @return {number[][]}
 */
var permute = function (nums) {
  const result = [];
  const used = new Array(nums.length).fill(false); // 标记元素是否已在当前路径中

  const backtrack = (path) => {
    // 终止条件：路径长度等于数组长度，说明找到一个全排列
    if (path.length === nums.length) {
      result.push([...path]); // 必须放副本，否则引用会被后续操作修改
      return;
    }

    for (let i = 0; i < nums.length; i++) {
      // 如果该元素已使用，跳过
      if (used[i]) continue;

      // 1. 做选择
      path.push(nums[i]);
      used[i] = true;

      // 2. 递归：进入下一层决策树
      backtrack(path);

      // 3. 撤销选择（回溯）：状态重置
      path.pop();
      used[i] = false;
    }
  };

  backtrack([]);
  return result;
};
```

### Combination Sum: 回溯中“元素可重复选取”的处理。

为了解决去重问题，我们在递归时引入一个 start 索引，确保每一层递归只能从当前数字或之后的数字中选择，从而避免回头选择导致重复。

```javascript
/**
 * @param {number[]} candidates
 * @param {number} target
 * @return {number[][]}
 */
var combinationSum = function (candidates, target) {
  const result = [];

  // 排序不是必须的，但排序后可以进行“剪枝”优化，提前结束不必要的搜索
  candidates.sort((a, b) => a - b);

  const backtrack = (remaining, path, start) => {
    // 1. 终止条件：如果剩余目标值为 0，说明找到了一组解
    if (remaining === 0) {
      result.push([...path]);
      return;
    }

    for (let i = start; i < candidates.length; i++) {
      // 2. 剪枝优化：如果当前数字已经大于剩余目标，由于数组已排序，后续数字肯定也大
      if (candidates[i] > remaining) break;

      // 3. 做选择
      path.push(candidates[i]);

      // 4. 递归：由于元素可以重复使用，下一层的 start 依然是 i，而不是 i + 1
      backtrack(remaining - candidates[i], path, i);

      // 5. 撤销选择（回溯）
      path.pop();
    }
  };

  backtrack(target, [], 0);
  return result;
};
```

## 7. 动态规划与分治 (6 题)

这是区分候选人水平的关键，重点在于状态转移方程。

### Climbing Stairs: 1D DP 的入门。

Climbing Stairs这是一个经典的**动态规划（Dynamic Programming）**问题。要到达第 $n$ 级台阶，你只有两种可能的前一步：从第 $n-1$ 级台阶跨 1 步。从第 $n-2$ 级台阶跨 2 步。因此，到达第 $n$ 级的总方案数 $f(n)$ 满足：$f(n) = f(n-1) + f(n-2)$。这本质上就是斐波那契数列。

```javascript
/**
 * @param {number} n
 * @return {number}
 */
var climbStairs = function (n) {
  // 基础情况处理
  if (n <= 2) return n;

  // 我们不需要存储整个数组，只需记录前两步的结果以优化空间
  let first = 1; // 到达第 1 级的方案数
  let second = 2; // 到达第 2 级的方案数

  for (let i = 3; i <= n; i++) {
    // 当前台阶的方案数 = 前一级 + 前两级
    let current = first + second;

    // 滚动更新变量
    first = second;
    second = current;
  }

  return second;
};
```

### House Robber: 状态转移方程的构建练习。

这又是一个经典的**动态规划（Dynamic Programming）**问题。面对第 $i$ 间房子，你只有两个选择：抢劫它：那么你就不能抢第 $i-1$ 间房，总金额 = 第 $i-2$ 间房的最大金额 + 当前房子的钱。不抢它：那么你可以维持抢完第 $i-1$ 间房时的最大金额。因此，状态转移方程为：$$dp[i] = \max(dp[i-1], dp[i-2] + nums[i])$$

```javascript
/**
 * @param {number[]} nums
 * @return {number}
 */
var rob = function (nums) {
  const n = nums.length;
  if (n === 0) return 0;
  if (n === 1) return nums[0];

  // 我们使用滚动变量来优化空间复杂度至 O(1)
  // prev2 代表 dp[i-2], prev1 代表 dp[i-1]
  let prev2 = 0;
  let prev1 = 0;

  for (const num of nums) {
    // 计算当前位置能拿到的最大值
    let current = Math.max(prev1, prev2 + num);

    // 更新滚动变量
    prev2 = prev1;
    prev1 = current;
  }

  return prev1;
};
```

### Coin Change: 完全背包问题的变体。

我们需要找到凑成目标金额所需的最少硬币数。核心思路是：要凑出金额 i，我们可以尝试减去每一个面值的硬币 coin。那么凑成 i 的最少硬币数，就等于凑成 i - coin 的最少硬币数加 1。

```javascript
/**
 * @param {number[]} coins - 硬币面值数组
 * @param {number} amount - 目标金额
 * @return {number} - 最少硬币个数
 */
var coinChange = function (coins, amount) {
  // 1. 初始化 dp 数组，dp[i] 表示凑齐金额 i 所需的最少硬币数
  // 我们将初始值设为 amount + 1，这相当于逻辑上的“无穷大”
  const dp = new Array(amount + 1).fill(amount + 1);

  // 2. 基础状态：凑齐金额 0 需要 0 个硬币
  dp[0] = 0;

  // 3. 外层循环遍历金额，内层循环遍历硬币
  for (let i = 1; i <= amount; i++) {
    for (let coin of coins) {
      // 如果当前金额比硬币面值大，说明可以使用这枚硬币
      if (i - coin >= 0) {
        // 状态转移方程：取当前值和 (用掉这枚硬币后的剩余金额所需的硬币数 + 1) 的最小值
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
      }
    }
  }

  // 4. 如果 dp[amount] 仍为初始值，说明无法凑齐
  return dp[amount] > amount ? -1 : dp[amount];
};
```

### Longest Increasing Subsequence: $O(n^2)$ 与 $O(n \log n)$ 两种解法的对比。

定义 $dp[i]$ 为以 $nums[i]$ 结尾的最长递增子序列的长度。为了计算 $dp[i]$，我们需要回头看索引 $i$ 之前的所有元素 $j$，如果 $nums[i] > nums[j]$，说明 $nums[i]$ 可以接在以 $nums[j]$ 结尾的序列后面。

```javascript
/**
 * @param {number[]} nums
 * @return {number}
 */
var lengthOfLIS = function (nums) {
  if (nums.length === 0) return 0;

  // 1. 初始化 dp 数组，每个元素自身就是一个长度为 1 的子序列
  const dp = new Array(nums.length).fill(1);
  let maxLen = 1;

  // 2. 遍历每一个元素作为子序列的终点
  for (let i = 1; i < nums.length; i++) {
    // 3. 检查当前点之前的所有点
    for (let j = 0; j < i; j++) {
      if (nums[i] > nums[j]) {
        // 如果当前数大，尝试更新 dp[i]
        dp[i] = Math.max(dp[i], dp[j] + 1);
      }
    }
    // 记录全局最大值
    maxLen = Math.max(maxLen, dp[i]);
  }

  return maxLen;
};
```

### Longest Palindromic Substring: 动态规划或中心扩展法。

解决最长回文子串最直观且高效的方法是中心扩展法（Expand Around Center）。回文串的特性是中心对称，因此我们可以遍历字符串中的每一个字符（以及每两个字符中间的间隙），将其作为中心向两边扩展。

```javascript
/**
 * @param {string} s
 * @return {string}
 */
var longestPalindrome = function (s) {
  if (s.length < 2) return s;

  let start = 0;
  let maxLength = 0;

  // 核心扩展函数
  const expand = (left, right) => {
    // 只要两侧字符相等，就继续向外扩展
    while (left >= 0 && right < s.length && s[left] === s[right]) {
      left--;
      right++;
    }
    // 回退一步后的实际长度为 (right - 1) - (left + 1) + 1 = right - left - 1
    return right - left - 1;
  };

  for (let i = 0; i < s.length; i++) {
    // 情况 1: 回文中心是一个字符，如 "aba" (i=1, left=1, right=1)
    let len1 = expand(i, i);
    // 情况 2: 回文中心是两个字符间隙，如 "abba" (i=1, left=1, right=2)
    let len2 = expand(i, i + 1);

    // 取两种情况下的最大长度
    let len = Math.max(len1, len2);

    if (len > maxLength) {
      maxLength = len;
      // 根据长度和中心点 i 计算起始索引
      // 注意：对于 len2，i 是左半部分的结尾，计算方式依然兼容
      start = i - Math.floor((len - 1) / 2);
    }
  }

  return s.substring(start, start + maxLength);
};
```

### Edit Distance: (必考 Hard) 字符串编辑的经典 DP。

我们需要计算将 word1 转换成 word2 的最少步数。核心思路是：通过比较两个字符串的末尾字符，将大问题拆解为子问题。

```javascript
/**
 * @param {string} word1
 * @param {string} word2
 * @return {number}
 */
var minDistance = function (word1, word2) {
  const m = word1.length;
  const n = word2.length;

  // dp[i][j] 代表 word1 的前 i 个字符转换成 word2 的前 j 个字符所需的最少操作数
  const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  // 1. 基础状态初始化
  // 如果 word2 为空，word1 转换成它需要全部删除
  for (let i = 0; i <= m; i++) dp[i][0] = i;
  // 如果 word1 为空，word1 转换成 word2 需要全部插入
  for (let j = 0; j <= n; j++) dp[0][j] = j;

  // 2. 状态转移
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (word1[i - 1] === word2[j - 1]) {
        // 如果当前字符相等，不需要任何操作，继承上一步的结果
        dp[i][j] = dp[i - 1][j - 1];
      } else {
        // 如果字符不等，考虑三种情况并取最小值：
        // dp[i-1][j-1] + 1 -> 替换操作
        // dp[i-1][j] + 1   -> 删除操作
        // dp[i][j-1] + 1   -> 插入操作
        dp[i][j] = Math.min(dp[i - 1][j - 1], dp[i - 1][j], dp[i][j - 1]) + 1;
      }
    }
  }

  return dp[m][n];
};
```

## 8. 二分查找与堆 (4 题)

处理大规模数据和最值的利器。

### Search in Rotated Sorted Array: 二分查找的变形应用。

虽然数组被旋转了，但如果我们在中点 mid 将其切开，一半必定是有序的，而另一半包含旋转点。

```javascript
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number}
 */
var search = function (nums, target) {
  let left = 0;
  let right = nums.length - 1;

  while (left <= right) {
    let mid = Math.floor((left + right) / 2);

    if (nums[mid] === target) return mid;

    // 核心逻辑：判断哪一半是有序的
    // 如果左边有序
    if (nums[left] <= nums[mid]) {
      // 检查 target 是否在左侧有序区间内
      if (nums[left] <= target && target < nums[mid]) {
        right = mid - 1;
      } else {
        left = mid + 1;
      }
    }
    // 否则，右边必定是有序的
    else {
      // 检查 target 是否在右侧有序区间内
      if (nums[mid] < target && target <= nums[right]) {
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }
  }

  return -1;
};
```

### Find First and Last Position of Element in Sorted Array: 二分查找寻找边界。

要在 $O(\log n)$ 的时间内解决这个问题，最直观的方法是进行两次二分查找：一次寻找目标值的“左边界”，一次寻找“右边界”。

```javascript
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
var searchRange = function (nums, target) {
  // 查找第一个等于 target 的位置
  const findFirst = (nums, target) => {
    let left = 0,
      right = nums.length - 1;
    let index = -1;
    while (left <= right) {
      let mid = Math.floor((left + right) / 2);
      if (nums[mid] >= target) {
        // 如果中间值大于等于目标，继续向左找，看前面还有没有
        right = mid - 1;
      } else {
        left = mid + 1;
      }
      if (nums[mid] === target) index = mid;
    }
    return index;
  };

  // 查找最后一个等于 target 的位置
  const findLast = (nums, target) => {
    let left = 0,
      right = nums.length - 1;
    let index = -1;
    while (left <= right) {
      let mid = Math.floor((left + right) / 2);
      if (nums[mid] <= target) {
        // 如果中间值小于等于目标，继续向右找，看后面还有没有
        left = mid + 1;
      } else {
        right = mid - 1;
      }
      if (nums[mid] === target) index = mid;
    }
    return index;
  };

  return [findFirst(nums, target), findLast(nums, target)];
};
```

### Kth Largest Element in an Array: 堆排序或快速选择（Quickselect）。

在快排中，每次 partition 后，基准值 (pivot) 都会落在它最终排序后的正确位置。如果这个位置正好是 $n-k$，我们就找到了答案。

```javascript
/**
 * @param {number[]} nums
 * @param {number} k
 * @return {number}
 */
var findKthLargest = function (nums, k) {
  const targetIndex = nums.length - k; // 第 k 大即为升序排序后的第 n-k 个

  const quickSelect = (left, right) => {
    const pivot = nums[right];
    let p = left;

    // 分区：将小于 pivot 的数移到左边
    for (let i = left; i < right; i++) {
      if (nums[i] <= pivot) {
        [nums[p], nums[i]] = [nums[i], nums[p]];
        p++;
      }
    }
    // 将 pivot 移到中间正确位置
    [nums[p], nums[right]] = [nums[right], nums[p]];

    // 递归逻辑
    if (p === targetIndex) return nums[p];
    if (p < targetIndex) return quickSelect(p + 1, right);
    return quickSelect(left, p - 1);
  };

  return quickSelect(0, nums.length - 1);
};
```

### Find Median from Data Stream: (高频 Hard) 对顶堆（双堆法）实时求中位数。

我们将数据分成两部分：

左半部分（较小的一半）：用一个 最大堆 (Max-Heap) 存储。堆顶是左半部分最大的数。

右半部分（较大的一半）：用一个 最小堆 (Min-Heap) 存储。堆顶是右半部分最小的数。

```javascript
class MedianFinder {
  constructor() {
    // 假设已经有实现好的 Heap 类
    // maxHeap 存储较小的一半，minHeap 存储较大的一半
    this.maxHeap = new MaxHeap();
    this.minHeap = new MinHeap();
  }

  /** * @param {number} num
   */
  addNum(num) {
    // 1. 先把数塞进左边的最大堆
    this.maxHeap.push(num);

    // 2. 维持平衡：左边最大的必须小于等于右边最小的
    // 把左边最大的挪到右边
    this.minHeap.push(this.maxHeap.pop());

    // 3. 维持数量：如果右边比左边多了，再挪回左边
    // 我们约定：左堆的数量可以等于或比右堆多 1
    if (this.minHeap.size() > this.maxHeap.size()) {
      this.maxHeap.push(this.minHeap.pop());
    }
  }

  /**
   * @return {number}
   */
  findMedian() {
    if (this.maxHeap.size() > this.minHeap.size()) {
      // 总数为奇数，中位数在左堆堆顶
      return this.maxHeap.peek();
    } else {
      // 总数为偶数，中位数是两个堆顶的平均值
      return (this.maxHeap.peek() + this.minHeap.peek()) / 2;
    }
  }
}
```
