---
title: Linux’ Shell 编程题
date: 2019-11-08 06:26:35
tags:
- linux
categories:
- Technical Notes
---

问题1：  
使用for循环在/test目录下通过随机小写10个字母加固定字符串test批量创建10个html文件，名称例如为：
```
[root@test test]# sh /server/scripts/test.sh  
[root@test test]# ls  
coaolvajcq_test.html qnvuxvicni_test.html vioesjmcbu_test.html
gmkhrancxh_test.html tmdjormaxr_test.html wzewnojiwe_test.html
jdxexendbe_test.html ugaywanjlm_test.html xzzruhdzda_test.html
qcawgsrtkp_test.html vfrphtqjpc_test.html
```

>答案:[generate_random.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/generate_random.sh)


问题2:  
请用至少两种方法实现;将以上文件名中的test全部改成oldgirl(用for循环实现),并且html改成大写。

>答案:  
[rename_1.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/rename_1.sh)  
[rename_2.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/rename_2.sh)

问题3：  
批量创建10个系统帐号test01-test10并设置密码（密码为随机8位字符串）。
>答案:[create_users.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/create_users.sh)

问题4:  
写一个脚本，实现判断10.0.0.0/24网络里，当前在线用户的IP有哪些（方法有很多）
>答案:[ping.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/ping.sh)

问题5:  
写一个脚本解决DOS攻击生产案例
提示：根据web日志或者或者网络连接数，监控当某个IP并发连接数或者短时内PV达到100，即调用防火墙命令封掉对应的IP，监控频率每隔3分钟。防火墙命令为：iptables -A INPUT -s 10.0.1.10 -j DROP。
>答案：[blockip.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/blockip.sh)

问题6:  
开发shell脚本分别实现以脚本传参以及read读入的方式比较2个整数大小。以屏幕输出的方式提醒用户比较结果。注意：一共是开发2个脚本。当用脚本传参以及read读入的方式需要对变量是否为数字、并且传参个数做判断。
>答案：[compare_2.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/compare_2.sh)
类似的求和：
[sum_1.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/sum_1.sh)

问题7:  
打印选择菜单，一键安装Web服务：

[root@testscripts]# sh menu.sh
1.[install lamp]
2.[install lnmp]
3.[exit]

pls input the num you want:

要求：

1、当用户输入1时，输出“startinstalling lamp.”然后执行/server/scripts/lamp.sh，脚本内容输出”lamp is installed”后退出脚本；
2、当用户输入2时，输出“startinstalling lnmp.” 然后执行/server/scripts/lnmp.sh输出”lnmp is installed”后退出脚本;
3、当输入3时，退出当前菜单及脚本；
4、当输入任何其它字符，给出提示“Input error”后退出脚本。
5、要对执行的脚本进行相关条件判断，例如：脚本是否存在，是否可执行等。

>答案：[menu.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/menu.sh)

问题8:  
1) 监控web服务是否正常，不低于3种监控策略。
2) 监控db服务是否正常，不低于3种监控策略。要求间隔1分钟，持续监控。
>答案：[monitor_web](https://github.com/bjdzliu/script/blob/master/shell/quiz1/monitor_web.sh)

问题9:  
面试及实战考试题：监控web站点目录（/var/html/www）下所有文件是否被恶意篡改（文件内容被改了），如果有就打印改动的文件名（发邮件），定时任务每3分钟执行一次(10分钟时间完成)。
>答案：[monitor_web_dir.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/monitor_web_dir.sh)

问题10:  
抓阄题目：运维派提供外出企业项目实践机会（第6次）来了（本月中旬），但是，名额有限，队员限3人（班长带队）。

因此需要挑选学生，因此需要一个抓阄的程序：

要求：

1、执行脚本后，想去的同学输入英文名字全拼，产生随机数01-99之间的数字，数字越大就去参加项目实践，前面已经抓到的数字，下次不能在出现相同数字。
2、第一个输入名字后，屏幕输出信息，并将名字和数字记录到文件里，程序不能退出继续等待别的学生输入

>答案：[chooseauser.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/chooseauser.sh)

问题11：  
bash for循环打印下面这句话中字母数不大于6的单词(昆仑万维面试题)。
I am test teacher welcome to test training class.
>答案：[charlength.sh](https://github.com/bjdzliu/script/blob/master/shell/quiz1/charlength.sh)
