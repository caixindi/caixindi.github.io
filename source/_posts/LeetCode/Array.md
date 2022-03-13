### [1.两数之和](https://leetcode-cn.com/problems/two-sum/description/)

|  Category  |  Difficulty   | Likes | Dislikes |
| :--------: | :-----------: | :---: | :------: |
| algorithms | Easy (50.92%) | 10826 |    -     |

<details open="" style="color: rgb(212, 212, 212); font-family: -apple-system, BlinkMacSystemFont, &quot;Segoe WPC&quot;, &quot;Segoe UI&quot;, system-ui, Ubuntu, &quot;Droid Sans&quot;, sans-serif, &quot;Microsoft Yahei UI&quot;; font-size: 14px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><summary><strong>Tags</strong></summary><p style="margin-top: 0px; margin-bottom: 0.7em;"><a href="https://leetcode.com/tag/array" title="https://leetcode.com/tag/array" style="color: var(--vscode-textLink-foreground); text-decoration: none;"><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">array</code></a><span>&nbsp;</span>|<span>&nbsp;</span><a href="https://leetcode.com/tag/hash-table" title="https://leetcode.com/tag/hash-table" style="color: var(--vscode-textLink-foreground); text-decoration: none;"><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">hash-table</code></a></p></details>

<details open="" style="color: rgb(212, 212, 212); font-family: -apple-system, BlinkMacSystemFont, &quot;Segoe WPC&quot;, &quot;Segoe UI&quot;, system-ui, Ubuntu, &quot;Droid Sans&quot;, sans-serif, &quot;Microsoft Yahei UI&quot;; font-size: 14px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><summary><strong>Companies</strong></summary><p style="margin-top: 0px; margin-bottom: 0.7em;"><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">adobe</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">airbnb</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">amazon</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">apple</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">bloomberg</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">dropbox</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">facebook</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">linkedin</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">microsoft</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">uber</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">yahoo</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">yelp</code></p></details>

给定一个整数数组 `nums` 和一个整数目标值 `target`，请你在该数组中找出 **和为目标值** 的那 **两个** 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。

**示例 1：**

```
输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。
```

**示例 2：**

```
输入：nums = [3,2,4], target = 6
输出：[1,2]
```

**示例 3：**

```
输入：nums = [3,3], target = 6
输出：[0,1]
```

**提示：**

- `2 <= nums.length <= 103`
- `-109 <= nums[i] <= 109`
- `-109 <= target <= 109`
- **只会存在一个有效答案**

------

[Discussion](https://leetcode-cn.com/problems/two-sum/comments/) | [Solution](https://leetcode-cn.com/problems/two-sum/solution/)

```java
import java.util.HashMap;
import java.util.Map;

/*
 * @lc app=leetcode.cn id=1 lang=java
 *
 * [1] 两数之和
 */

// @lc code=start

class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> hashtable = new HashMap<Integer, Integer>();
        for (int i = 0; i < nums.length; i++) {
            //判断哈希表中是否存在目标元素，如果存在，直接返回当前遍历位置以及目标元素value组成的数组
            if (hashtable.containsKey(target - nums[i])) {    
                return new int[] { hashtable.get(target - nums[i]), i };
            }
            //不存在，将当前元素加入到哈希表中
            hashtable.put(nums[i], i);
        }
        return new int[0];
    }
}
// @lc code=end
```





### [寻找两个正序数组的中位数](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/description/)

|  Category  |  Difficulty   | Likes | Dislikes |
| :--------: | :-----------: | :---: | :------: |
| algorithms | Hard (39.94%) | 3986  |    -     |

<details open="" style="color: rgb(212, 212, 212); font-family: -apple-system, BlinkMacSystemFont, &quot;Segoe WPC&quot;, &quot;Segoe UI&quot;, system-ui, Ubuntu, &quot;Droid Sans&quot;, sans-serif, &quot;Microsoft Yahei UI&quot;; font-size: 14px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><summary><strong>Tags</strong></summary><p style="margin-top: 0px; margin-bottom: 0.7em;"><a href="https://leetcode.com/tag/array" title="https://leetcode.com/tag/array" style="color: var(--vscode-textLink-foreground); text-decoration: none;"><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">array</code></a><span>&nbsp;</span>|<span>&nbsp;</span><a href="https://leetcode.com/tag/binary-search" title="https://leetcode.com/tag/binary-search" style="color: var(--vscode-textLink-foreground); text-decoration: none;"><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">binary-search</code></a><span>&nbsp;</span>|<span>&nbsp;</span><a href="https://leetcode.com/tag/divide-and-conquer" title="https://leetcode.com/tag/divide-and-conquer" style="color: var(--vscode-textLink-foreground); text-decoration: none;"><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">divide-and-conquer</code></a></p></details>

<details open="" style="color: rgb(212, 212, 212); font-family: -apple-system, BlinkMacSystemFont, &quot;Segoe WPC&quot;, &quot;Segoe UI&quot;, system-ui, Ubuntu, &quot;Droid Sans&quot;, sans-serif, &quot;Microsoft Yahei UI&quot;; font-size: 14px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><summary><strong>Companies</strong></summary><p style="margin-top: 0px; margin-bottom: 0.7em;"><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">adobe</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">apple</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">dropbox</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">google</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">microsoft</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">yahoo</code><span>&nbsp;</span>|<span>&nbsp;</span><code style="color: var(--vscode-textPreformat-foreground); font-family: var(--vscode-editor-font-family, &quot;SF Mono&quot;, Monaco, Menlo, Consolas, &quot;Ubuntu Mono&quot;, &quot;Liberation Mono&quot;, &quot;DejaVu Sans Mono&quot;, &quot;Courier New&quot;, monospace); font-size: 1em; line-height: 1.357em; white-space: pre-wrap;">zenefits</code></p></details>

给定两个大小分别为 `m` 和 `n` 的正序（从小到大）数组 `nums1` 和 `nums2`。请你找出并返回这两个正序数组的 **中位数** 。

 

**示例 1：**

```
输入：nums1 = [1,3], nums2 = [2]
输出：2.00000
解释：合并数组 = [1,2,3] ，中位数 2
```

**示例 2：**

```
输入：nums1 = [1,2], nums2 = [3,4]
输出：2.50000
解释：合并数组 = [1,2,3,4] ，中位数 (2 + 3) / 2 = 2.5
```

**示例 3：**

```
输入：nums1 = [0,0], nums2 = [0,0]
输出：0.00000
```

**示例 4：**

```
输入：nums1 = [], nums2 = [1]
输出：1.00000
```

**示例 5：**

```
输入：nums1 = [2], nums2 = []
输出：2.00000
```

 

**提示：**

- `nums1.length == m`
- `nums2.length == n`
- `0 <= m <= 1000`
- `0 <= n <= 1000`
- `1 <= m + n <= 2000`
- `-106 <= nums1[i], nums2[i] <= 106`

 

**进阶：**你能设计一个时间复杂度为 `O(log (m+n))` 的算法解决此问题吗？

------

[Discussion](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/comments/) | [Solution](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/solution/)

```java
//方法一
import java.util.ArrayList;
/*
 * @lc app=leetcode.cn id=4 lang=java
 *
 * [4] 寻找两个正序数组的中位数
 */

// @lc code=start
class Solution {
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        ArrayList<Integer> nums3=new ArrayList<Integer>();
        int i=0,j=0;
        int m=nums1.length;
        int n=nums2.length;
        for (int k = 0; i<m&&j<n&&k < m+n-1; k++) {
                if (nums1[i] < nums2[j])
                    nums3.add(nums1[i++]);
                else 
                    nums3.add(nums2[j++]);
            }
            if(i==m)
                for(;j<n;j++)
                    nums3.add(nums2[j]);
            else
                for(;i<m;i++)
                    nums3.add(nums1[i]);
        int s=m+n;
        double result=s%2==1?nums3.get(s/2):(nums3.get(s/2-1)+nums3.get(s/2))/2.0;
        return result;
    }
}
// @lc code=end


//方法二
/*
 * @lc app=leetcode.cn id=4 lang=java
 *
 * [4] 寻找两个正序数组的中位数
 */

// @lc code=start
class Solution {
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        int i=0,j=0;
        int m=nums1.length;
        int n=nums2.length;
        if((m+n)%2==1){
            for (int k = 0; i<m&&j<n&&k <=(m+n)/2; k++){
                if(k==(m+n)/2){
                    if (nums1[i] < nums2[j])
                        return nums1[i]/1.0;
                    else
                        return nums2[j]/1.0;
                }
                if (nums1[i] < nums2[j])
                    i++;
                else 
                    j++;
            }
            if(j==n)
                return nums1[(m+n)/2-j]/1.0;
            else
                return nums2[(m+n)/2-i]/1.0;
        }else{
            for (int k = 0; i<m&&j<n&&k <=(m+n)/2; k++){
                if(k==(m+n)/2-1){
                    if(i<m-1&&j<n-1){
                    if ((nums1[i] < nums2[j])&&(nums1[i+1]<nums2[j]))
                        return (nums1[i]+nums1[i+1])/2.0;
                    else if((nums1[i] <= nums2[j])&&(nums1[i+1]>=nums2[j]))
                        return (nums1[i]+nums2[j])/2.0;
                    else if((nums1[i] > nums2[j])&&(nums1[i]>nums2[j+1]))
                        return (nums2[j]+nums2[j+1])/2.0;
                    else if((nums1[i] > nums2[j])&&(nums1[i]<=nums2[j+1]))
                        return (nums2[j]+nums1[i])/2.0;
                    }else if(i==m-1&&j!=n-1){
                        if (nums1[i] <= nums2[j]){
                            return (nums1[i]+nums2[j])/2.0;
                        }else{
                            if(nums2[j+1]<=nums1[i])
                                return (nums2[j]+nums2[j+1])/2.0;
                            else
                                return (nums1[i]+nums2[j])/2.0;
                        }

                    }else if(j==n-1&&i!=m-1){
                        if (nums2[j] <=nums1[i]){
                            return (nums1[i]+nums2[j])/2.0;
                        }else{
                            if(nums2[j]<=nums1[i+1])
                                return (nums1[i]+nums2[j])/2.0;
                            else
                                return (nums1[i]+nums1[i+1])/2.0;
                        }
                    }else if(i==m-1&&j==n-1){
                        return (nums1[i]+nums2[j])/2.0;
                    }
                }
                if (nums1[i] < nums2[j])
                    i++;
                else 
                    j++;
            }
            if(j==n)
                return (nums1[(m+n)/2-j-1]+nums1[(m+n)/2-j])/2.0;
            else
                return (nums2[(m+n)/2-i-1]+nums2[(m+n)/2-i])/2.0;
        }
    }
}
// @lc code=end

```

