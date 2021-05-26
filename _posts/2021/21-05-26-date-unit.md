---
title: UTC, GMT, KST 표시
categories: 
tags:
   
---
javascript가 편하게 알려준다.
### format
- ISO
```javascript
(new Date).toISOString() 
```
```javascript
// 현재 세계 표준 시간: 5월26일 1시27분
// 현재 한국 기준 시간: 5월26일 10시27분
"2021-05-26T01:27:24.177Z"
```
- UTC,GMT
```javascript
(new Date).toUTCString()
or
(new Date).toGMTString()
```
```javascript
// 현재 세계 표쥰 시간: 5월26일 1시31분
// 현재 한국 기준 시간: 5월26일 10시31분
"Wed, 26 May 2021 01:31:37 GMT"
```
- KST(Local)
```javascript
new Date
```
```javascript
// 현재 한국 기준 시간: 5월26일 10시33분
Wed May 26 2021 10:33:16 GMT+0900 (Korean Standard Time)
```



