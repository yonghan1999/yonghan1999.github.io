---
layout: post
title: 第 196 场周赛
date: 2020-07-05
categories: 技术
tags: LeetCode
---

### 5452. 判断能否形成等差数列

~~~java
class Solution {
    public boolean canMakeArithmeticProgression(int[] arr) {
        Arrays.sort(arr);
        if(arr.length==1)
            return true;
        int c=arr[1]-arr[0];
        for(int i=1;i<arr.length-1;i++) {
            if(arr[i+1]-arr[i]!=c)
                return false;
        }
        return true;
    }
}
~~~

### 5453. 所有蚂蚁掉下来前的最后一刻

~~~java
class Solution {
    public int getLastMoment(int n, int[] left, int[] right) {
        int max=0;
		for(int i=0;i<right.length; i++) {
			int max_t = n-right[i];
			max = Math.max(max, max_t);
		}
		for(int i=0;i<left.length; i++) {
			int max_t = left[i];
			max = Math.max(max, max_t);
		}
		return max;
    }
}
~~~

### 5454. 统计全 1 子矩阵

解题思路
矩阵里每个点(i.j)统计他这行左边到他这个位置最多有几个连续的1，存为left[i][j]。然后对于每个点(i.j)，我们固定子矩形的右下角为(i.j)，利用left从该行i向上寻找子矩阵左上角为第k行的矩阵个数。每次将子矩阵个数加到答案中即可。
时间复杂度O(nnm)，空间复杂度O(nm)。

~~~java
class Solution {
public:
    int numSubmat(vector<vector<int>>& mat) {
        int n = mat.size();
        int m = mat[0].size();
        vector<vector<int> > left(n,vector<int>(m));
        int now = 0;
        for(int i=0;i<n;i++){
            now = 0;
            for(int j=0;j<m;j++){
                if(mat[i][j] == 1) now ++;
                else now = 0;
                left[i][j] = now;
            }
        }
        int ans = 0,minx;
        for(int i=0;i<n;i++){
            for(int j=0;j<m;j++){
                minx = 0x3f3f3f3f;
                for(int k=i;k>=0;k--){
                    minx = min(left[k][j],minx);
                    ans += minx;
                }
            }
        }
        return ans;
    }
};

作者：lin-miao-miao
链接：https://leetcode-cn.com/problems/count-submatrices-with-all-ones/solution/5454-tong-ji-quan-1-zi-ju-xing-by-lin-miao-miao/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
~~~