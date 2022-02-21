---
layout: post
title: iOS 提示框
date: 2021-07-09
categories: 技术
tags: iOS
---
### iOS 中弹出一个提示框

~~~objective-c
//要显示的信息
NSString *message = [[NSString alloc] initWithFormat:@"message",(long)indexPath.section,(long)indexPath.row];
//初始化并设置一个 title 和 message
UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"title" message:message preferredStyle:UIAlertControllerStyleAlert];
//设置一个按钮并指定按钮的 title 和处理函数。
[alert addAction:[UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
    //点击确定后执行的操作；
      NSLog(@"你点击了确定");
  }]];
  [self presentViewController:alert animated:true completion:^{
    //显示提示框后执行的事件；
      NSLog(@"此时出现提示");
      
  }];
~~~

