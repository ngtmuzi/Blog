---
title: mingw32缺少posix_memalign函数    
desc: 瞎玩 
date: 2015-11-30 10:40:47.000
tags: C++
author: ngtmuzi  
category: 神秘代码  
---

想在`windows`上研究`word2vec`，于是查了一下如何`make`之类的东西，装好`mingw32`之后编译的时候总是报错说找不到这个函数，查了半天百度，结论大概就是没有，不过有类似的函数`_aligned_malloc`，但是参数格式不一样，那还是自己动手吧：
```c
function posix_memalign( void ** memptr, size_t alignment, size_t size){
  (* memptr) = _aligned_malloc(size, alignment);
}
```

这样就可以编译通过了