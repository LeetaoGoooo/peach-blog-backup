---
title: 基于 Ajax 请求的 Flask-Login 认证
tag: [Flask,Flask Web 开发,Python]
comments: true
date: 2019-01-03
---


# 前言

关于 Flask-Login 大家肯定不陌生,之前我还写过一篇关于 [Flask-Login 用户管理](https://zhuanlan.zhihu.com/p/30530494)有兴趣的可以看一下,在 Flask 的场景中,我们对 Flask-Login 的使用,通常都是像下面代码这样:

```python
@auth.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(user_name=form.user_name.data).first()
        if user is not None and user.verify_password(form.password.data):
            login_user(user, form.remember_me.data)
            return redirect(url_for('main.index'))
        flash('邮箱或者密码错误!')
    return render_template('auth/login.html', form=form)
```

一个登录表单,然后 login_user 结束战斗,这个时候关于用户的相关信息就保存在 cookies 中了,然后在其他地方就可以开心愉快的调用了 current_user 了,那么问题了来了,有的时候我们不想通过通过 cookie 的话,该如何处理呢?

# User_Loader 孪生兄弟 -- Request_Loader

在之前的文章中有提到 user_loader, 这里也不说了. 和 user_loader 一样,我们同样需要实现 request_loader 的回调函数, 不一样的是, 之前通过传 user 标识 ID,这里是传 Flask 的 request 请求


## 通过 api_key 登录认证

```python
@login_manager.request_loader
def load_user_from_request(request):
    api_key = request.args.get('api_key')
    # 校验 api_key
    if api_key:
        user = User.query.filter_by(api_key=api_key).first()
        if user:
            return user
```

## 通过 Authorization header 登录认证

```python
@login_manager.request_loader
def load_user_from_request(request):
    authorization = request.headers.get('Authorization')
    # 校验 authorization
    # ... 省略
```


说完上面俩种校验方式,我们的重头戏来了,如何通过 ajax 来完成这一认证操作呢?

# Login with ajax over Flask-Login

关于 api_key 和 Authorization 的获取,通常情况下会约定,或者在登录的时候统一返回,然后可以保存在 localStorage或者 cookie 中,每次请求的时候获取动态添加到 ajax 请求中

## ajax with api_key

```javascript
$.ajax({
            type: "get",
            url: 'xxxx?api_key=api_key',
            contentType: "application/json;charset=utf-8",
            success: function (data) {
                //
            },error:function(error){
                //
            }
    });
```

## ajax with Authorization header

```javascript
// post
$.ajax({
            type: "post",
            url: '',
            contentType: "application/json;charset=utf-8",
            data :JSON.stringify({"test":"test"}),
            dataType: "json",
            beforeSend: function (XMLHttpRequest) {
         	    XMLHttpRequest.setRequestHeader("Authorization", "Basic test");
            },
            success: function (data) {
                // 
            },error:function(error){
                //
            }
    });

// get
$.ajax({
            type: "get",
            url: '',
            contentType: "application/json;charset=utf-8",
            beforeSend: function (XMLHttpRequest) {
         		XMLHttpRequest.setRequestHeader("Authorization", "Basic test");
            },
            success: function (data) {
                // 
            },error:function(error){
                //
            }
    });
```

# 完整例子

目录结构

```
templates/
    -----index.html
example.py
```

## index.html

```index.html
<!DOCTYPE html>
<html>
    <head>
        <title> test login by Leetao</title>
    </head>
    <body>
        <p id="test_login"></p>
        <button onclick="load_msg()">未点击</button>
        <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/1.12.4/jquery.min.js"></script>
        <script>
            var load_msg = function () {
                $.get('/hello?api_key=test_login',function(data){
                    $('#test_login')[0].innerText = data
                })
            }
        </script>
    </body>
</html>
```

## example.py

```python
from flask import Flask, request, jsonify, render_template
from flask_login import LoginManager, current_user, login_required
login_manager = LoginManager()

app = Flask(__name__)
login_manager.init_app(app)

class User:
    
    def __init__(self,user_name):
        self.id = 'test_id'
        self.user_name = user_name

    @property
    def is_active(self):
        return True

    @property
    def is_authenticated(self):
        return True

    @property
    def is_anonymous(self):
        return False

    def get_id(self):
        try:
            return text_type(self.id)
        except AttributeError:
            raise NotImplementedError('No `id` attribute - override `get_id`')

user = User("leetao")

@login_manager.request_loader
def load_from_request(request):
    api_key = request.args.get('api_key')
    if api_key == 'test_login':
        return user
    return None

@app.route('/hello')
@login_required
def hello_world():
    print(current_user.user_name)
    return jsonify('Hello, World!')

@app.route("/")
def index():
    return render_template("index.html")
```

## 结果

为了方便理解,我截了两张图,一张是 api_key 正确的情况下,一张是 错误的情况下

### api_key 正确

![](http://ww1.sinaimg.cn/large/006wYWbGly1fytiw435blj30zh0g740b.jpg)

### api_key 错误

![](http://ww1.sinaimg.cn/large/006wYWbGly1fytiwcjbcej30vk0i1abi.jpg)