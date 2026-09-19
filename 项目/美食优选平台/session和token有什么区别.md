+ session 服务端保存用户登录状态，客户端只保存 sessionid
	+ 本项目是基于redis的共享session
+ token 客户端保存认证信息，服务端通过验证 token 来识别用户，通常使用jwt
	+ 缺点：jwt 一旦签发，在过期前不好主动失效。