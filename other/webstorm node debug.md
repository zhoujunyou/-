### webstorm node debug

[链接](https://cnodejs.org/topic/571605522de81867132a793f)

#### 使用superagent设置cookie

```javascript
superagent
                .get('http://mnr.test.cn/index.php/shop/Webaccount/getlist')
                .query({userName:'mwtydzpdwp2'})
                .set('Cookie', 'sso9nowcn_auth=a6af8iM58fo%2BC8HLaALQsFGyBEvisuMshdyQZWyrFXCSb%2F0%2BPnwkyVx1PTHo')
                .end(function (err, res2) {
                     console.log(res2)
                })
```



####使用superagent获取cookie

```javascript
superagent
                .post('http://oa.mwbyd.cn/SSO/Login/check_login.htm')
                .type('form')
                .send({
                    mobile:'15901724016',
                    password: '123qweasd',
                    remember: 'on'
                })
                .end(function (err, res2) {
                    if(err){
                       return next(err)
                    }
                    let cookie=res2.header['set-cookie'])
                    res.send(res2.text)
                })
```



#### 比较js时间大小

```javas
Date.parse('01/01/2011 10:20:45') > Date.parse('01/01/2011 5:10:10')
```



### Async/Await替代Promise的6个理由[跳转](https://blog.fundebug.com/2017/04/04/nodejs-async-await/)

[为什么我的await 不生效](https://stackoverflow.com/questions/38428027/why-await-is-not-working-for-node-request-module):因为返回的Promis才能用



