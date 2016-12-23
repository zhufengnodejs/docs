## cookie
HTTP1.0中协议是无状态的，但在WEB应用中，在多个请求之间共享会话是非常必要的，所以出现了Cookie  
cookie是为了辩别用户身份，进行会话跟踪而*存储在客户端*上的数据
## Cookie的处理流程
<img src="http://7xjf2l.com1.z0.glb.clouddn.com/cookies.png" class="img-responsive">

## 使用步骤
1. 客户端第一次访问服务器的时候服务器通过响应头向客户端发送Cookie,属性之间用分号空格分隔
```javascript
Set-Cookie:name=zfpx; Path=/
```
2. 客户端接收到Cookie之后保存在本地
<img src="http://7xjf2l.com1.z0.glb.clouddn.com/localcookie.jpg" class="img-responsive">

3. 以后客户端再请求服务器的时候会把此Cookie发送到服务器端
```javascript
Cookie:name=zfpx
```

## 重要属性
|属性|说明|
|:-------|:-------|
|name=value|键值对，可以设置要保存的 Key/Value|
|Domain|域名，默认是当前域名|
|maxAge|最大失效时间(毫秒),设置在多少后失效|
|secure|当 secure 值为 true 时，cookie 在 HTTP 中是无效，在 HTTPS 中才有效|
|Path|表示 cookie 影响到的路，如 path=/。如果路径不能匹配时，浏览器则不发送这个Cookie|
|Expires|过期时间(秒)，在设置的某个时间点后该 Cookie 就会失效，如 expires=Money, 05-Dec-11 11:11:11 GMT|
|httpOnly|如果在COOKIE中设置了`httpOnly`属性，则通过程序(JS脚本)将无法读取到COOKIE信息，防止XSS攻击产生|

## 向客户端发送`cookie`
### 设置cookie
```
 res.cookie(name,value,[,options]);
```
|参数|chrome对应属性|类型|说明|示例|
|:-------|:-------|:-------|:-------|:-------|
|domain|Domain|String|域名，默认是当前域名|{domain:'a.zfpx.cn'}|
|path|Path|String|路径，默认是/|{path:'/visit'}|
|expires|Expires|Date|过期时间，如果没以有指定或为0表示当前会话有效|{expires:new Date(Date.now()+20*1000)}|
|maxAge|Max-Age|Number|有效时间(单位是毫秒)|{maxAge:20*1000}|
|httpOnly|HTTP|Boolean|不能通过浏览器javascript访问|{httpOnly:true}|
|secure|Secure|String|只通过https协议访问| |

## 获取cookie
使用cookie-parser中间件
```javascript
$ npm install cookie-parser --save
```

```javascript
app.use(require('cookie-parser')());    //使用中间件
response.cookie(key,value)              //在响应中向客户端设置cookie
request.cookies                         //获取请求中的cookie对象
response.clearCookie('username')        //清除cookie  
```

## 记录客户端的访问次数

```javascript
var express = require('express');
var cookieParser = require('cookie-parser');
var app = express();
/**
 * 如果要加密的话 cookieParser里要指定密码，而且signed要等于true
 */
app.use(cookieParser('zfpx'));
app.get('/write',function(req,res){
    //1.普通设置
    //res.cookie('name','value');

    //2.设置域名
    //res.cookie('name','zfpx',{domain:'a.zfpx.cn'});

    //3.设置路径
    //res.cookie('name','zfpx',{path:'/visit'});

    //4.过期时间
    //res.cookie('name','zfpx',{expires:new Date(Date.now()+20*1000)});//毫秒
    //res.cookie('name','zfpx',{maxAge:20*1000});//过期时间 毫秒

    //httpOnly true还是false无意义 document.cookie取不到
    //res.cookie('name','zfpx',{httpOnly:true});
    res.cookie('age','123',{signed:true});
    res.end('ok');
});

app.get('/read',function(req,res){
    console.log(req.signedCookies);
    res.send(req.cookies);
});

//记录这是客户端的第几次访问
app.get('/visit',function(req,res){
    res.cookie('count',isNaN(req.cookies.count)?0:parseInt(req.cookies.count)+1);
    res.send(req.cookies);
});


app.listen(9090);
```

## cookieParser原理解析
```javascript
function cookieParser(req, res, next){
    if (!req.headers.cookie) {
        return next();
    }
    req.cookies =  require('querystring').parse(req.headers.cookie,'; ','=');
    res.cookie = cookie;
    next();
}

function cookie(name, val, options) {
    var opt = options || {};

    var value = encodeURIComponent(val);

    var pairs = [name + '=' + value];

    if (null != opt.maxAge) {
        var maxAge = opt.maxAge - 0;
        if (isNaN(maxAge)) throw new Error('maxAge should be a Number');
        pairs.push('Max-Age=' + Math.floor(maxAge));
    }

    if (opt.domain) {
        pairs.push('Domain=' + opt.domain);
    }

    if (opt.path) {
        pairs.push('Path=' + opt.path);
    }

    if (opt.expires) pairs.push('Expires=' + opt.expires.toUTCString());
    if (opt.httpOnly) pairs.push('HttpOnly');
    if (opt.secure) pairs.push('Secure');

    return pairs.join('; ');
}
```

## 加密cookie
```javascript
var crypto = require('crypto');
var sign = function(val, secret){
    return val + '.' + crypto
            .createHmac('sha256', secret)
            .update(val)
            .digest('base64')
            .replace(/\=+$/, '');
};

console.log(sign('123','zfpx'));
```

## 权限控制
```javascript
var express = require('express');
var cookieParser = require('cookie-parser');
var app = express();
app.set('view engine','html');
app.engine('html',require('ejs').__express);
app.set('views',__dirname);

app.use(cookieParser());

function checkUser(req,res,next){
    if(req.cookies && req.cookies.username)
      next();
    else
      res.redirect('/');
}

//进入登录页
app.get('/',function(req,res){
    res.render('index');
});

//登录
app.get('/login',function(req,res){
    res.cookie('username',req.query.username,{httpOnly:true});
    res.redirect('/user');
});

//用户页面
app.get('/user',checkUser,function(req,res){
    res.render('user',{username:req.cookies.username});
});

//用户退出
app.get('/logout',function(req,res){
    res.clearCookie('username');//清除cookie
    res.redirect('/');
});


app.listen(8080);
```

## 记住密码
```javascript
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script>
        function setCookie(key,value){
            document.cookie = key+"="+value;
        }
        function getCookie(key){
            if(document.cookie){
                var parts = document.cookie.split('; ');
                for(var i=0;i<parts.length;i++){
                    var vals = parts[i].split('=');
                    if(vals[0]==key){
                        return vals[1];
                    }
                }
                return "";
            }else{
                return "";
            }
        }
        function checkCookie(){
            var username = getCookie('username');
            if(username){
                alert('欢迎'+username+'再次访问');
                document.querySelector('#username').value = username;
            }else{
                username = prompt('请输入用户名:');
                if(username){
                    setCookie('username',username);
                }
            }
        }
    </script>
</head>
<body onLoad="checkCookie()">
<input type="text" id="username">
</body>
</html>
```
## cookie使用注意事项

- 可能被客户端篡改，使用前验证合法性
- 不要存储敏感数据，比如用户密码，账户余额
- 使用httpOnly保证安全
- 尽量减少cookie的体积
- 设置正确的domain和path，减少数据传输
