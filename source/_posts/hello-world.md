---
title: Hello World
date: 2024-12-12 12:00:00
tags: [test]
cover: "/imgs/wallhaven-exmo98.jpg"
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

> this is a quote

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

code test
```c++
class Solution {
public:
    void maxHeapify(vector<int>& nums, int i, int heapSize) {
        int left = 2 * i + 1, right = 2 * i + 2;
        int largest = i;
        if(left < heapSize && nums[left] > nums[largest]) {
            largest = left;
        }
        if(right < heapSize && nums[right] > nums[largest]) {
            largest = right;
        }
        if(largest != i) {
            swap(nums[i], nums[largest]);
            maxHeapify(nums, largest, heapSize);
        }
    }

    void buildMaxHeap(vector<int>& nums, int heapSize) {
        for(int i = heapSize / 2 - 1; i >= 0; i--) {
            maxHeapify(nums, i, heapSize);
        }
    }

    int findKthLargest(vector<int>& nums, int k) {
        int heapSize = nums.size();
        buildMaxHeap(nums, heapSize);
        for(int i = nums.size() - 1; i >= nums.size() - k + 1; i--) {
            swap(nums[0], nums[i]);
            --heapSize;
            maxHeapify(nums, 0, heapSize);
        }
        return nums[0];
    }
};
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)
