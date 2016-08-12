---
title: popToViewController的坑
tags:
- iOS
---

在使用popViewController时候遇到了两个比较隐蔽的问题.因此,在以后的开发中需要自己注意.


### tips1
在调用popViewController时,使用GCD丢到main queue中去执行:

``` objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
	...
   [self.navigationController popViewControllerAnimated:NO];
	...
});
```

### tips2

在代码中可能连续多次调用`popViewControllerAnimated`地方,最好通过遍历`navigationController.viewControllers`找到具体要遍历到哪个再直接pop到目标`controller`.

``` objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    BOOL findIt = NO;
    UIViewController *targetVC = nil;
    for (UIViewController *subVC in self.navigationController.viewControllers) {
        if (findIt) {
            break;
        }
        if (subVC == xxx) {
            findIt = YES;
        }else{
            targetVC = subVC;
        }
    }
	[self.navigationController popToViewController:targetVC animated:NO];

});

```