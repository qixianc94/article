## 爬取有js “YunSuoAutoJump()” 的网站 

疫情原因，工作不多，突然想尝试爬取一个完整的门户网站，在慕课网看到有爬取“伯乐在线”，自己试了试，发现伯乐在线加了反爬虫机制。
### 第一次爬取
```
>>scrapy shell http://jobbole.com/xinwen/xwyd/91760.html
>>response.text
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
	<meta http-equiv="Cache-Control" content="no-store, no-cache, must-revalidate, post-check=0, pre-check=0"/>
	<meta http-equiv="Connection" content="Close"/>
	<script type="text/javascript">
		function stringToHex(str){
			var val="";
			for(var i = 0; i < str.length; i++){
				if(val == "")val = str.charCodeAt(i).toString(16);
				else val += str.charCodeAt(i).toString(16);
			}
				return val;
		}
		function YunSuoAutoJump(){
			var width =screen.width; 
			var height=screen.height; 
			var screendate = width + "," + height;
			var curlocation = window.location.href;
			if(-1 == curlocation.indexOf("security_verify_")){
				document.cookie="srcurl=" + stringToHex(window.location.href) + ";path=/;";
				console.log(document.cookie)
			}
			self.location = "/xinwen/xwyd/91760.html?security_verify_data=" + stringToHex(screendate);}
	</script>
	<script>setTimeout("YunSuoAutoJump()", 50);</script>
</head><!--2020-02-28 11:20:05-->
</html>
```
这里没有返回网页内容，直接返回了一个js，发现YunSuoAutoJump()做了两件事：  
1.设置cookie  
2.重定向到xxxx+security_verify_data=" + stringToHex(screendate)
请求带回的cookie
`security_session_verify=666abeb963acbe117a0f1ef91e833989`
### 第二次爬取
那就模拟js代码设置cookie和继续访问新的连接，这里改用requests，调试简单
```
import requests
def stringToHex(str):
	val = ""
	for i in range(len(str)):
		val += hex(ord(str[i]))[2:]
	return val
headers = {
    "User-Agent":"Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36"
}
base_url = "http://jobbole.com/xiaofei/wj/2606.html"
cookie = {"srcurl":stringToHex(base_url),"path":"/"}
# 第一次访问，为了获取cookie
response = requests.get(base_url,headers=headers)
# 存储cookie
for key, value in response.cookies.items():
	cookie[key] = value

#第二次访问，带上cookies，访问重定向的页面
response = requests.get(base_url+"?security_verify_data=323536302c31343430",headers=headers,cookies=cookie)
print(response.content)
```
第二次请求还是不正确，和之前有区别的是，js和cookie都不一样了
```
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
	<meta http-equiv="Cache-Control" content="no-store, no-cache, must-revalidate, post-check=0, pre-check=0"/>
	<meta http-equiv="Connection" content="Close"/>
	<script>
		var cookie_custom = {
			hasItem: function (sKey) {
				return (new RegExp("(?:^|;\\\\s*)" + encodeURIComponent(sKey).replace(/[\\-\\.\\+\\*]/g, "\\\\$&") + "\\\\s*\\\\=")).test(document.cookie);   
			},
			removeItem: function (sKey, sPath) {
	  			if (!sKey || !this.hasItem(sKey)) {
	   				return false; 
				} 
				document.cookie = encodeURIComponent(sKey) + "=; expires=Thu, 01 Jan 1970 00:00:00 GMT" + ( sPath ? "; path=" + sPath : "");
				return true; 
			}
		};
		function YunSuoAutoJump() {
			self.location = "/"; 
		}
	</script>
	<script>setTimeout("cookie_custom.removeItem(\'srcurl\');YunSuoAutoJump();", 50);</script>
	</head><!--2020-02-28 11:34:21-->
</html>'
```
`security_session_mid_verify=cfe17a5313a84921504130d58beb15ed `
`security_session_verify=0a8703b8590265c1bf6e234722922dbf`
能看到这次js做的事情:  
1.删除'srcurl'这个cookie
2.重定向到'/'


### 第三次爬取
加判断，如果有'removeItem'那就删除cookie的'srcurl'，并访问重定向页面  
多试几次第二次爬取的代码，就会发现这种js重定向页面只有两样，/和base_url  
针对这一种js，写出代码
```
...
    if 'srcurl' in cookie:
        del cookie['srcurl']
    # 正则找出url
    s = re.search('location = "(.*?)"',response.text).group(1)
    if s == '/':
        response = requests.get("http://jobbole.com"+s,headers=headers,cookies=cookie)
    else:
        response = requests.get(base_url,headers=headers,cookies=cookie)
```
这时候得到的response就是我们想要的网站内容了


### 最后加上逻辑判断
最后网站内容是没有location关键词的，就用这个来终止循坏。然后爬的太快ip就被禁用了，加上sleep
```
# 完整代码
import requests
import time
import re
def stringToHex(str):
	val = ""
	for i in range(len(str)):
		val += hex(ord(str[i]))[2:]
	return val
headers = {
    "User-Agent":"Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36"
}
base_url = "http://jobbole.com/gongsi/qywh/84717.html"
response = requests.get(base_url,headers=headers)
cookie = {"srcurl":stringToHex(base_url),"path":"/"}
# 存储cookie
for key, value in response.cookies.items():
	cookie[key] = value
	

while 'location' in response.text:
	time.sleep(1)
	# 简单的判断是哪种js
	if "stringToHex" in response.text:
		response = requests.get(base_url+"?security_verify_data=323536302c31343430",headers=headers,cookies=cookie)
		print(base_url+"?security_verify_data=323536302c31343430")
		print(response.text)
	else:
		if 'srcurl' in cookie:
			del cookie['srcurl']
		s = re.search('location = "(.*?)"',response.text).group(1)
		if s == '/':
			response = requests.get("http://jobbole.com"+s,headers=headers,cookies=cookie)
		else:
			response = requests.get(base_url,headers=headers,cookies=cookie)
	for key, value in response.cookies.items():
		cookie[key] = value
# 存起来，已经拿到内容了，随便操作
with open('blogs.txt','w',encoding="utf-8") as f:
	f.write(response.content.decode())
```
