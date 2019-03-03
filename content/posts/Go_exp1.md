---
title: "Go web 编程踩过的坑"
date: 2019-03-03T15:58:24+08:00
draft: true
tags: ["技术"]
summary: "最简单开始的最弱智错误. "
---
最近想用Go写一个web后端，所以先从最简单的开始
目录架构
```
project-----
        |---tmeplates---home.html
        |                           
        |---static----
        |           |---style.css
        |           |---xxx.js
        |-server.go
```
html 部分
```
<!DOCTYPE html>
<html>
<head>
<link rel="stylesheet" type="text/css" href="static/style.css">

<title>My Page</title>
</head>
<body>
<p>The date today is {{.Date}}</p>
<p>And the time is {{.Time}}</p>
</body>
</html>
```
server.go  部分
```
package main

import (
	"fmt"
	"html/template"
	"log"
	"net/http"
	"time"
)

func helloWorld(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "hellow World")

}

type PageVariables struct {
	Date string
	Time string
}

func HomePage(w http.ResponseWriter, r *http.Request) {
	now := time.Now()
	HomePageVars := PageVariables{
		Date: now.Format("02-01-2006"),
		Time: now.Format("15:04:05"),
	}
	t, err := template.ParseFiles("templates/home.html")
	if err != nil {
		log.Print("template parsing error: ", err)
	}

	err = t.Execute(w, HomePageVars)
	if err != nil {
		log.Print("template executing error: ", err)
	}

}

func main() {
	http.HandleFunc("/", HomePage)
	http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("static"))))
	http.ListenAndServe(":8080", nil)

}

/*Refference 参考
https://blog.scottlogic.com/2017/02/28/building-a-web-app-with-go.html
*/

```
非常简单，html里面会用到static文件夹里面的css而已，但是这个就是坑！
这个是由下面这行code实现的
```
http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("static"))))

```
坑就在于开始的时候我将```http.Dir("static")``` 写成了```http.Dir("/static")``` .没错就是多了个反斜杠```/```,但是这会让Go在根目录下去找```static```文件夹
可以这么理解，```http.FileServer(http.Dir("static"))``` 就是将当前的```static```文件夹当成是文件服务器，也就是我们可以通过http访问文件夹的内容（官方api解释：[FileServer returns a handler that serves HTTP requests with the contents of the file system rooted at root.](https://golang.org/pkg/net/http/#FileServer)）, 但是我们并不想直接在```127.0.0.1:8080```就是对应到这个静态文件夹，而是对应到```127.0.0.1:8080/static```，所以我们用```http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("static"))))```处理, ```http.StripPrefix```是将```\static\```这个prefix去掉，因为现在static文件夹就是相当于我们的根目录。

最后是一些参考：
[StripPrefix](https://golang.org/pkg/net/http/#StripPrefix)
[How do I serve CSS and JS in Go Lang](https://stackoverflow.com/questions/43601359/how-do-i-serve-css-and-js-in-go-lang)
[building-a-web-app-with-go](https://blog.scottlogic.com/2017/02/28/building-a-web-app-with-go.html)