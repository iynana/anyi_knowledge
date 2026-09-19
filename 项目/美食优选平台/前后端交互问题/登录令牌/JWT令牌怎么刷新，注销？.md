双token模式：
- 刷新：客户端携带 refresh token 请求新 access token，服务端验证 refresh token 有效性（未过期、未吊销），生成新 access token 返回。
- 注销：
	- 直接删除refresh token
	- 服务端维护黑名单（Redis），将需注销的 access token 加入黑名单（存储至原过期时间），每次请求校验黑名单。refresh token 也可加入黑名单。

refresh token是一个随机字符串，存储在redis中 `refresh:token:{tokenId} -> userInfo`
access token是jwt，jwt的结构是三段，用 `.` 分隔：
```
Header.Payload.Signature
```
例如：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMDAxIiwianRpIjoiYWJjLTEyMyIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAwOTAwfQ
.
signature
```
三部分含义：
```
Header：说明使用什么签名算法
Payload：存放用户和过期时间等声明
Signature：防篡改签名
```