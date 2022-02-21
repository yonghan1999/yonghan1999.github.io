---
layout: post
title: Foundation Framework
date: 2021-07-07
categories: 技术
tags: Objective-c
---

### NSArray

~~~objective-c
//创建一个 NSArray
NSArray *colors = @[@"Red", @"Yellow", @"Orange",
                         @"Green", @"Blue", @"Violet"];
NSArray *numbers = @[@6,@2,@3,@4,@5];
NSArray *cities = [NSArray arrayWithObjects:@"New Delhi",
                    @"London", @"Brisbane", @"Adelaide", nil];
//遍历
for (NSString *item in colors) {
    NSLog(@"%@", item);
}
//普通遍历
for (int i=0; i<[colors count]; i++) {
    NSLog(@"%d: %@", i, colors[i]);
}
// 简单排序
// 自带的compare:方法，也可以自己写compare:方法
[numbers sortedArrayUsingSelector: @selector(compare:)] // 升序 2，3，4，5，6
//简单排序，反转使其改为降序
[[[numbers sortedArrayUsingSelector: @selector(compare:)] reverseObjectEnumerator] allObjects]//降序 6，5，4，3，2
//查找某个元素，结返回布尔值，存在返回 True，不存在返回 False；
[colors containsObject:@"Red"]
//查找一个元素，若存在返回下标，若不存在返回NSNotFound
NSUInteger index = [colors indexOfObject:@"Red"];
//比较两个数组是否相等，每对元素都是同个 isEqual 进行测试的
[arr1 isEqualToArray:arr2]

//创建一个 NSMutableArray
NSMutableArray * arrM = [NSMutableArray array];
//添加一个元素
[arrM addObject:@"cwj"];
//将一个 NSArray 添加进一个数组中
[arrM addObjectsFromArray:@[@"abc",@"def"]];
//插入一个元素到指定下标
[arrM insertObject:@"ghi" atIndex:1];



~~~

### NSString、NSMutableString

~~~objc
//初始化一个NSString，stringWithFormat 是个不错的选择
NSString *sample = @"iOS Tutorials";
NSString *message = [NSString stringWithFormat:@"That's a %@ %@ from %d!",@"qq", @"message", 2021];


// isEqualToString 比较两个字符串是否相等, hasPrefix 检查前缀, hasSuffix 检查后缀。
NSString *strName = @"Programming Language";
if ([strName isEqualToString:@"Programming Language"]) {
    NSLog(@"The name string holds the text Programming Language");
}
if ([strName hasPrefix:@"Programming"]) {
    NSLog(@"The first name of the word is Programming");
}
if ([strName hasSuffix:@"Language"]) {
    NSLog(@"The second name of the word is Language");
}


//拼接两个或者多个字符串
NSLog(@"Combine two strings：%@", [sample stringByAppendingString:message]);
NSLog(@"Combine multiple strings：%@", [NSString stringWithFormat:@"%@%@%@",@"String1",@"String2",@"String3"]);

//字符串查询 查询字符串 text 中是否包含字符串 s；
NSString *text = @"this is a string";
NSString *s = @"is";
NSRange searchResult = [text rangeOfString:s];
if (searchResult.location == NSNotFound) {
    NSLog(@"Search string was not found");
} else {
    NSLog(@"location %lu length %lu",
          searchResult.location,		//2		???????????? 为什么是 2 ？？？？
          searchResult.length);			//2
}



// 可变 String ???????   使用 setString 对内容进行替换
NSMutableString *result = [NSMutableString stringWithString:@"Mutable String Text"];
[result setString:@"Modified String"];
NSRange range = [result rangeOfString:@" String"];
//删除
[result deleteCharactersInRange:range]; //Modified
//插入
[result insertString:@"insert" atIndex:range.location];
//替换
[result stringByReplacingOccurrencesOfString:@"Modified" withString:@"modified"];  //??????????想实现替换但是这个代码并没有起作用

//将 NSMutableString 转换成 NSString
NSString *t = [[result copy] autorelease];

~~~

### NSNumber

~~~objc
// NSNumber 常用的数据类型以及使用方法
NSNumber *aBool = @NO;
NSNumber *aChar = @'z';
NSNumber *anInt = @2147483647;
NSNumber *aUInt = @4294967295U;
NSNumber *aLong = @9223372036854775807L;
NSNumber *aFloat = @26.99F;
NSNumber *aDouble = @26.99;
NSLog(@"aBool: %@\n aChar: %@\n anInt: %@\n aUInt: %@\n aLong: %@\n aFloat: %@ aDouble: %@",aBool,aChar,anInt,aUInt,aLong,aFloat,aDouble);
~~~



