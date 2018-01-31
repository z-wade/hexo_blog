title: ThreeSum 的 swift实现
date: 2016-07-03 23:15:45
tags: 算法

---

第一次刷leetcode，觉得收获挺多的

<!-- more -->

```

class Solution {
    func threeSum(nums: [Int]) -> [[Int]] {
        var result:[[Int]] = [];
        if nums.count < 3 {
            return result;
        }
        
        var array = bubbleSort(nums)
        for i in 0 ..< array.count-2 {
            if i != 0 && array[i] == array[i-1] {
                continue
            }
            if array[i] > 0 {
                break;
            }
            var j = i + 1;
            var r = array.count - 1;
            while j < r {
                let sum = array[i] + array[r] + array[j]
                if sum == 0 {
                    result.append([array[i],array[j],array[r]]);
                    print("\(array[i]) + \(array[j]) + \(array[r]) = \(sum)")
                    
                    // print("\(array[j]) 与 \(array[j+1])");
                    while j < r && array[j] == array[j+1] {
                        j += 1
                    }
                    while j < r && array[r] == array[r-1] {
                        r -= 1
                    }
                    j += 1
                    r -= 1
                }else if(sum < 0 ){
                    j += 1
                    
                }else{
                    r -= 1
                }
            }
        }
        return result
    }
    
    func bubbleSort(nums: [Int]) -> [Int] {
        var  result = nums;
        
        for i in 0 ..< result.count{
            for j  in 0 ..< result.count-i-1 {
                if result[j] > result[j+1] {
                    let temp = result[j]
                    result[j] = result[j+1]
                    result[j+1] = temp
                }
            }
            // print(result);
        }
        return result;
    }
}

```

求和为0的话，比较容易。难点是怎么去重，一开始思路错了，导致后面去重很麻烦..