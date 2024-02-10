---
title: HGAME Week1
toc: true
date: 2024-01-31 02:19:11
categories: 技术
cover: 安全
---
# HGAME
出于一些机缘巧合，暑假里加入了[Vidar-Team](https://vidar.club/)的新生群，逐渐开始了解CTF，八月打了一段时间的hgame-mini，一度（指大佬还没来的时候）排到了第一页  
虽然加入vidar基本没有可能，但也有幸认识了一些成员，尤其是[Answer](https://4nsw3r.top/)和[eking](https://ek1ng.com/)对我帮助非常大，也很感谢[baimeow](https://baimeow.cn/)和[potato](https://potat0.cc/)带我加入了[DN11](https://dn11.top/)，对计网有了点更深入的了解  
开学后各种事情以及开发任务逐渐增加，同时也很难找到队友，甚至于参加省赛时三个人的队伍剩下两个人一道题都没做出来，我只能选择放弃了安全这个方向。杭电的学长们一直在提的HGAME这会儿开始了，也就在寒假里抽空随便做点吧，应该是~~根本不存在的~~生涯中最后一赛了

# MISC
## 签到
![](https://s11.ax1x.com/2024/01/31/pFKRdu8.png)
```
hgame{welc0m3_t0_HGAME_2024}
```
到此一游

## SignIn
![try_another_way_to_see.png]
传到手机上从不同方向看
```
hgame{WOW_GREAT_YOU_SEE_IT_WONDERFUL}
```

# Web
## ezHTTP
> 前半段貌似hgame-mini同款

访问靶机，返回`请从vidar.club访问这个页面`
设置请求Headers
```
Referer: vidar.club
```
返回`请通过Mozilla/5.0 (Vidar; VidarOS x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 Edg/121.0.0.0访问此页面`，接着改Headers
```
User-Agent: Mozilla/5.0 (Vidar; VidarOS x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 Edg/121.0.0.0
```
返回`请从本地访问这个页面`，设置`X-Forwarded-For: 127.0.0.1`，发现返回值没有变化，仔细一看返回Headers里有个`Hint: Not XFF`  
![](https://s11.ax1x.com/2024/01/31/pFKR0Hg.png)
略加思索~~（指ChatGPT）~~，把XFF改成了`X-Real-IP`，返回值变成了`Ok, the flag has been given to you ^-^`  
返回Headers里有`Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJGMTRnIjoiaGdhbWV7SFRUUF8hc18xbVAwclQ0bnR9In0.VKMdRQllG61JTReFhmbcfIdq7MvJDncYpjaT7zttEDc`，一眼JWT的格式，base64解码之后得到flag
```
hgame{HTTP_!s_1mP0rT4nt}
```
## Bypass it
访问靶机，弹窗提示`欢迎使用用户管理系统，请先登陆`，点击确定后跳转到`/login.html`
![](https://s11.ax1x.com/2024/01/31/pFKRbgx.png)
点击注册按钮后跳转到`register_page.php`，虽然会弹窗`很抱歉，当前不允许注册`  
开始还以为题目名字的意思是SQL注入，试了好久也没成功，后来无意中发现注册页面可以看到元素  
``` html
<li>
	<label>用户名:</label>
	<input type="text" name="username" />
</li>
<li>
	<label>密　码:</label>
	<input type="password" name="password" />
</li>
```
根据表单构造请求，返回数据
``` html
<script language='javascript' defer>
    alert('注册成功');top.location.href='login.html'
</script>
```
正常登录即可获取到flag
![](https://s11.ax1x.com/2024/01/31/pFKROKK.png)
```
hgame{b5619c116a434cda50308d84f381ea85124d1b2e}
```

## Select Courses
打开靶机，啊这正方教务的UI……
![](https://s11.ax1x.com/2024/01/31/pFKRrNj.png)
所有课都显示已满，点击有弹窗提示`课程已满！`
![](https://s11.ax1x.com/2024/01/31/pFKRs4s.png)
打开开发者工具，发现点击选课按钮的时候会POST请求`/api/courses`，payload为`{"id":1}`，返回`{"full":1,"message":"课程已满"}`  
点右上选完了按钮会请求`/api/ok`并返回`{message: "呜呜呜，还没选上课呢！"}`  
查看源码还可以发现课程列表来自GET请求`/api/courses`  
然后……最近项目有点多，只能腾出这半小时，没时间研究了，就这样吧（
> 后来看wp，原来是思路完全错了，要轮询抢课