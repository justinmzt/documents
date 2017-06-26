# mongoose
## 先安装Node.js的mongoose库

npm install mongoose


## db.js
创建db.js

``` javascript
var mongoose = require('mongoose');
//连接数据库
db = mongoose.connect('mongodb://localhost:27017/mongo');
//定义Schema（表结构）
var userSchema = new mongoose.Schema({
    name     : String,
    password : String
});
//创建model
var model = db.model('User', userSchema);
//封装方法
function save(query, callback){
    var user = new model(query);
    user.save(callback);
}
module.exports = {
    save: save
};
```
## handler.js

``` javascript
var User = require('./db');

function save(response) {
    User.save({
        name: 'admin',
        password: 'admin'
    },function(err,user){
        response.writeHead(200, {"Content-Type": "text/html"});
        response.write(JSON.stringify(user));
        response.end();
    })
}

exports.save = save;
```
如果顺利，页面上可以看到JSON的字符串。