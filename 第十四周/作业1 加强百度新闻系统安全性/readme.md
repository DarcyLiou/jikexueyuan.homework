#对百度新闻做安全的优化
**项目采用node.js + restify + orm + mysql, 项目的使用请参看<a href="#guide">使用前简要说明</a> **  
**关于安全增强优化的部分，请参考<a href="#security">安全增强做的优化</a>**

##<a name="guide">使用前简要说明</a>
1. 入口文件为index.js 主要是路由配置和错误处理。
2. data_module文件夹下保存的是配置文件和本系统使用的模块。
3. /documents保存的是前端文件，包括百度新闻手机页面(/documents/client),新闻管理后台(/documents/server)
4. /data_module/db_module.js 文件中定义了orm数据模型。
5. /data_module/sys_config.js 文件中保存了一些本系统使用的配置，包括对数据库名称，数据库连接配置文件的配置，具体使用方法可参考该文件中的注释。
6. 这次使用Nodejs做后台的主要思路是不改变前端使用的接口，因此接口保持了一致没有对前端做修改。本来使用restify可以把路由设置的更加REST化。

##入口页面及路由说明
默认的入口页面设置：  

|系统|入口页面|
|---|---|
|百度新闻手机版| http://[server]:[port]/ |
|新闻管理后台| http://[server]:[port]/server/ 注意:一定要加上最后的“/” |

路由设置说明：
```javascript
// 新闻写入相关操作, 通过POST参数action来区分增加/修改/删除操作
server.post('/api/news/update/', ...)

// 获取新闻相关操作, 包括通过id，类目，页码等获取操作， 通过query字符串参数action区分。
server.get('/api/news/', ...)

// 获取所有类目信息
server.get('/api/category/categories', ...) 

// 类目写入相关操作，通过POST参数action来区分增加/修改/删除操作
server.post('/api/category/update', ...)

// 初始化数据库
server.post('/db/init', ...)

// 系统设置了404跳转，对于跳转页面设置了静态文件服务路由。
server.get(/\/errorpage\/{1,}.*/, ...)

// 管理后台的静态文件服务路由设置，设置了默认文件，可以通过 http://[server]:[port]/server来访问
server.get(/\/server\/{1,}.*/, ...)

// 前端百度手机新闻页面路由，设置了默认文件，可以通过http://[server]:[port]/来访问
server.get(/\/?.*/, ...)
```

##依赖项
1. restify  主要使用restify作为页面的路由设置。  
2. orm2     主要使用orm来访问数据库。
restify使用了dtrace-provider模块，但是由于drace-provider模块在安装时有错误，我这里的处理是将dtrace-provider模块删除。

##几点疑问
1. 使用orm，感觉对于数据对象化的操作比较直观。但是感觉通过对象之间的操作无法实现像sql那样的复杂操作。  
例如：获取所有类目的同时，返回该类目下新闻的个数。这个操作  
使用sql，采用存储过程，基本是两条sql解决问题。  
使用orm，则需要先获取所有类目，然后每条类目获取该类目下所有的新闻数。感觉翻译成sql就是许多条。  
实际使用感觉如果使用ORM，一个简单的操作可能会被翻译成多条SQL，增加了数据库的负担。
2. 

##开发环境
###软件版本
|软件|操作系统|nodejs|MySql|
|---|---|---|---|---|
|版本|OS X 10.11.2|v4.2.3|mysql-5.7.10-osx10.9-x86_64|

##使用nodejs遇到的问题。
1. nodejs module可以在服务器运行期间维持module内部的状态。而其他语言如php，.net不可以，如果需要页面间维持状态需要客户端回传状态参数或者使用去服务器级的全局变量。个人理解这个对后端开发有着完全不同的思路，更像是在开发桌面应用，而不同于传统基于页面的后端开发。
2. nodejs 的基于事件的非阻塞特点，与传统的阻塞性的语言不同。在程序结构和处理思路上也有不同。nodejs优先使用事件回调。

##使用restify遇到的问题。 
1. 使用serveStatic模块时，服务的文件夹路径不能是根路径。  
```javascript
server.get(/\/?.*/, restify.serveStatic({   
    directory: '.',  //注意这里不能是'.'，否则会提示没有权限。   
    default: 'index.html'  
}));  
```

2. 使用serverStatic需要注意，服务路径为正则表达式组合directory的路径.  
```javascript
server.get(/\/docs\/?.*/, restify.serveStatic({
    directory: './documents/client',//注意这里服务的路径是./documents/client/docs/
    default: 'index.html'
}));
```

3. server.get/post方法的匹配顺序是先注册，先匹配，后注册，后匹配。

4. 通过restify获取POST数据，需要使用bodyParser模块，这个模块已经内置在restify中。  
在代码中加入如下内容，就可以回调函数中通过request.params来获取post回来的数据。下面的示例是官方给出的配置示例。
```javascript
server.use(restify.bodyParser({
    maxBodySize: 0,
    mapParams: true,
    mapFiles: false,
    overrideParams: false,
    multipartHandler: function(part) {
        part.on('data', function(data) {
          /* do something with the multipart data */
        });
    },
    multipartFileHandler: function(part) {
        part.on('data', function(data) {
          /* do something with the multipart file data */
        });
    },
    keepExtensions: false,
    uploadDir: os.tmpdir(),
    multiples: true,
    hash: 'sha1'
 }));
```
如果不使用客户端回传文件，可以简化为如下的方式：
```javascript
server.use(restify.bodyParser({
    maxBodySize: 0,
    mapParams: true, //关键这句表明了需要从body中映射参数，即POST参数。
    mapFiles: false, //这里关闭了POST文件的选项，因此相关文件的其他的选项可以不写。
    overrideParams: false, //这个是如果body中的传参的名称与GET传参名称相同，则覆盖。这个可以根据自己的情况修改
}));
```

5. restify获取query字符串参数，需要使用queryParser这个内置模块。
queryParser模块使用操作如下。就可以回调函数中通过request.params来获取query字符串的数据。
```javascript
server.use(restify.queryParser());
```

6. restify的静态文件服务，在路由配置时，建议要加上尾部的‘/’  
```javascript
// “/\/server\/{1,}.*/”该正则表达式要求路由的未必至少有1个“/”，推荐用这总方式来服务静态文件。
// “/\/server\/?.*/” 这是官网建议的方式，但是官网推荐方式存在下述问题：
// 该方式会存在如果用户访问不带有最后的“/”时，CSS加载的当前路径不是server，而是server的上一级。
// 这样服务的静态页面中的当前所有的相对路径都会向上一级。导致错误。
server.get(/\/server\/{1,}.*/, ...);
```

##使用orm遇到的问题。
1. 感觉ORM的逻辑对于数据库操作来说较为频繁。并不如直接使用SQL来的简便和高效。  
比如一个修改操作，在ORM中需要先找到这个模型对象，然后再删除它，对应ORM翻译过来是两个SQL。
2. 使用ORM可以有效的避免很多直接使用SQL的问题，比如SQL注入问题。
3. ORM可以对数据进行基本的操作，比较完善的数据过滤操作。但是不能实现较为复杂的操作。


##<a name="security">安全增强做的优化：</a>
###存储性的XSS防御
之前版本主要存在的是存储型的XSS问题，问题有两个  
- 一个是input框中的输入会原封不动的展示在网页上。
- 新闻录入本身就是html代码，直接保存在数据库中，有可能被篡改。
  
所做的优化如下：
- 针对input框的输入，因为这部分的内容预期不是html代码，因此对input框的输入在后端做了html的转义。  
    好处：展现的时候采用html展现，减轻了前端的转义压力，简化前端的显示操作，可保证输入与展现一致。  
    遇到的问题：当做修改的时候，依然展现在input中，但是input的value不支持html显示，因此将input修改为span，增加了可编辑属性。
    后端代码位置：/data_modules/db_operator.js line 16;
    
- 针对新闻的内容录入，采用了html白名单的处理，在后端进行过滤。过滤非预期的html代码，以保证内容的安全。  
    好处：可以有效限制非法的录入问题。  
    遇到的问题：增加了html过滤操作，增加了服务器的负担，好在新闻录入不是大量的操作。白名单设置如果不当，依然会造成XSS的注入可能。
    代码位置：/data_modules/db_operator.js line 16;
    
- 只在服务器端进行修改，而不是在前端修改。
    好处：系统架构采用的是Ajax数据交互方式，这会在前端代码中暴露ajax的接口，非法用户可以用数据伪造的方式来跳过前端代码进行攻击，因此在服务器端统一做了XSS的防御，而不是在前端。
    
###反射型的XSS
服务器系统建立在restify的基础上，而用户get请求的内容均会被转化为服务器参数，非法字符串会无法被转化为参数。  
另一方面，通过在入参上加强限制的方式来规避可能的字符。
代码位置：/data_modules/db_operator.js line 129,134,139。

###SQL注入
服务器的数据库链接使用的是ORM的方式，框架本身对于SQL注入提高了难度。（限于能力，我自己没有测试出来。）
目前经过上面的处理，还有一处**可能**出现安全问题是搜索部分，这部分为了防止SQL注入，也采用了HTML转义方式。
之所以采用HTML转义方式的原因是，保存在数据库中的合法输入均为HTML转义后的代码。
代码位置：/data_modules/db_operator.js line 144。

###几个问题
1. 我理解XSS的注入最为有效的是存储型的。
2. 反射型的XSS/DOM型的XSS对于钓鱼来说可能比较实用。用户一般只看网址，网址是对的，对于后面的参数用户一般是不理解的。
3. 变异型的，只是利用了上面两种情况的非健全的转码方式。使得原本合法的输入编程了XSS。
4. SQL注入其实就是通过用户输入构造非法的SQL。
5. 我理解，没有密不透风的墙，只是突破这个强需要的成本（时间，知识能力，资金等成本。）
