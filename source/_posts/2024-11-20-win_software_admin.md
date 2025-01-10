---
layout: post
title: 为软件添加打开时请求管理员权限
date: 2024-11-20
categories: 技术
tags: Windows 管理员权限
---


在项目开发的过程中，可能会遇到需要使用`Admin`权限的操作。第一种操作是可以要求软件使用者，通过鼠标右键菜单，选择“以管理员身份运行”方式来运行程序，这种方式虽然对软件开发者的要求降低了，但却对软件使用者的要求提高了，并不是一种妥当的方式。下面提供一种亲测可行的方式，将软件提升管理员`Admin`权限。

### 实现步骤

1. 新建一个文本文档，填入以下内容后，保存为`filename.manifest`（注意这里的filename可以是任意名字）：

   ~~~xml
   <?xml version='1.0' encoding='UTF-8' standalone='yes'?>
   <assembly xmlns='urn:schemas-microsoft-com:asm.v1' manifestVersion='1.0'>
     <trustInfo xmlns="urn:schemas-microsoft-com:asm.v3">
       <security>
         <requestedPrivileges>
           <requestedExecutionLevel level='requireAdministrator' uiAccess='false' />
         </requestedPrivileges>
       </security>
     </trustInfo>
   </assembly>
   ~~~

2. 如果没有安装Windows SDK，需要先安装[Windows SDK]([Windows SDK - Windows app development | Microsoft Developer](https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/))。并配置到PATH环境变量中。

3. 使用`cmd`执行，命令中`"filename.manifest"`是前面我们新建的`filename.manifest`的路径 ，`"example.exe"`是要授予管理员权限的程序的路径。

   ~~~cmd
    mt.exe -manifest "filename.manifest" -outputresource:"example.exe";#1
   ~~~

   输出以下结果，表明操作成功。

   ![在这里插入图片描述](https://image.hanblog.fun/images/2025/01/10/35fda38946280bab310458d603e3deb3.png)

4. 完成后程序上会出现盾牌标志，打开时会提示需要管理员权限。

   ![在这里插入图片描述](https://image.hanblog.fun/images/2025/01/10/69fa5ba116637daaa823287041037d52.png)
