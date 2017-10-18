# koa2-tutorial
基于koa2定制开发规范及中间件
# Node.js log 中间件指南

## 业务场景

一个真实的项目想要在线上正常运转，需要各种后续操作来集中记录线上问题，换言之，版本迭代和后期维护在其中起到了至关重要的作用。人工操作耗时耗力，因此为了更高效地发现和跟踪问题，我们会想到建立自动排查和跟踪机制。日志就是实现这种机制的关键。完善的日志记录不仅能够还原问题场景，还有助于统计访问数据，分析用户行为。

## 记录日志的目的 

* 监控服务的运行
* 帮助开发者分析和排查问题
* 结合监控系统（如 ELK ）给出预警

## 关于编写 log 中间件的预备知识

### log4js

本项目中的 log 中间件是基于 log4js 2.x 的封装，[Log4js](https://github.com/nomiddlename/log4js-node)是 Node.js 中记录日志是否成熟的第三方模块，下文也会根据中间件的使用介绍一些 log4js 的使用方法。	

### 日志分类

日志可以大体上分为访问日志和异常日志。访问日志一般记录客户端对项目的访问，主要是 http 请求；异常日志用来记录项目本身无法正常运行的情况，方便开发人员定位 bug ，从而快速修复项目。
	
### 日志等级

log4js 中的日志输出可分为如下7个等级：

![LOG_LEVEL.957353bf.png](http://upload-images.jianshu.io/upload_images/3860275-7e8db4f9d1aed430.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当定义了某个输出日志级别，会输出级别相等或更高级别的日志。比如：自定义显示错误级别为 error，那么会显示调用 error、fatal、mark 方法打印的日志。

### 日志切割

当我们的项目在线上环境稳定运行后，访问量会越来越大，日志文件也会越来越大。日益增大的文件对查看和跟踪问题带来了诸多不便，同时增大了服务器的压力。虽然可以按照类型将日志分为两个文件，但并不会有太大的改善。所以我们按照日期将日志文件进行分割。比如：今天将日志输出到 task-2017-10-16.log 文件，明天会输出到 task-2017-10-17.log 文件。减小单个文件的大小不仅方便开发人员按照日期排查问题，还方便对日志文件进行迁移。


## 如何实现

### middleware/index.js

将初始化 log 中间件的过程置于 http-error 中间件之后，便可以接收到除 http-error 中间件以外的所有中间件的日志记录，并使用全局监听进行错误处理。此外，使用 http-error 中间件对 log 中产生的 http 相关的错误进行捕捉和展示。

```
 /**
   * 初始化log模块
   */
   
  // 引入log中间件
  const miLog = require('./mi-log')

  module.exports = (app) => {
	
    ...http-error中间件
  
    app.use(miLog({
      env: app.env,		// 当前环境变量
      category: 'xxxxx',		// 自定义的分类名称
      projectName: 'node-tutorial', // 项目名称
      appLogLevel: 'debug',	// 显示log级别
      dir: 'logs',			// 自定义输出log文件夹
      serverIp: ip.address()  // 服务器IP
    }));
	
	...其他中间件
		
	/**
    * 全局监听错误事件
    */
    app.on('error', (err, ctx) => {
      if (ctx) {
        ctx.status = 500;
      }
      if (ctx && ctx.log && ctx.log.error) {
        ctx.status = 500;
        if (!ctx.state.logged) {
           ctx.log.error(err.stack);
        }
      }
    })

}
```

传入参数讲解：

*	env -- 用于设置模块内的环境
*	category -- 设置一个 Logger 实例的类型
*	projectName -- 用于记录项目名称，便于追踪
*	appLogLevel -- 定义需要打印的日志级别
*	dir -- 根据开发者需求定义文件夹名称
*	serverIp -- 记录日志存在的服务器

### middleware/mi-log/logger.js 

log 中间件的核心内容

```
   
  // 引入工具模块 
  const log4js = require('log4js');
  const path = require("path");
  const client = require("./client.js");

	// ALL OFF 这两个等级并不会直接在业务代码中使用
	const methods = ["trace", "debug", "info", "warn", "error", "fatal", "mark"];
	
	// 定义传入参数的默认值
	const baseInfo = {
	  appLogLevel: 'debug',
	  dir: 'logs',
	  category: 'default',
	  env: 'local',
	  projectName: 'default',
	  serverIp: '0.0.0.0'
	}

```

按照 log4js 2.x 文档中定义的日志输出格式


```
  const appenders = {
  	 task: {
	    type: 'dateFile',	 // 日志类型
	    filename: `${dir}/task`, // 输出文件名
	    pattern: '-yyyy-MM-dd.log',  //后缀
	    alwaysIncludePattern: true  // 是否总是有后缀名
	 }
  };
```

*  type：'dataFile' -- 按照日期的格式分割日志
*  filename -- 将自定义的输出目录与定义的文件名进行拼接
*  pattern -- 按照日期的格式每天新建一个日志文件

进行判断，若是开发环境，将日志同时打印到终端，方便开发者查看。

```
	if (env === "dev" || env === "local" || env === "development") {
   	appenders.out = {
      		type: "console"
    	}
	}
```

```
 const config = {
    appenders,
    categories: {
      default: {
        appenders: Object.keys(appenders),
        level: appLogLevel
      }
    }
  }
  const logger = log4js.getLogger(category);
  
  // ALL OFF 这两个等级并不会直接在业务代码中使用
  const methods = ["trace", "debug", "info", "warn", "error", "fatal", "mark"];

  const currentLevel = methods.findIndex(ele => ele === appLogLevel)



  // 将 log 挂在上下文上
  return async (ctx, next) => {

    log4js.configure(config);
    // level 以上级别的日志方法
    methods.forEach((method, i) => {
      if (i >= currentLevel) {
        contextLogger[method] = (message) => {
          logger[method](client(ctx, message, commonInfo))
        }
      } else {
        contextLogger[method] = () => {}
      }
    });
    ctx.log = contextLogger;
    await next()
  };

```

初始化 logger 函数，并返回异步函数。调用 log 函数打印日志时，循环 logger 中的方法将高于自定义级别的方法挂载到上下文上，并将低于自定义级别的方法赋值空函数。此外，不挂载到上下文上，暴露 logger 方法并在需要使用的地方按照模块化引入也是一种不错的选择。

### middleware/mi-log/client.js

为了获取上下文中的请求信息和客户端信息，我们增加一个 client 文件， 将其一同打印到日志信息中：

```
module.exports = (ctx, message, commonInfo) => {
  const {
    method,
    url,
    host,
    headers
  } = ctx.request;
  const client = {
    method,                             // 请求方式
    url,                                // 请求url
    host,                               // 请求方域名
    message,                            // 打印的错误信息
    referer: headers['referer'],        // 请求的源地址
    userAgent: headers['user-agent']    // 客户端信息采集
  }

  return JSON.stringify(Object.assign(commonInfo, client));
}
```

### middleware/mi-log/index.js

最后，调用 logger 主函数时再封装一层错误处理，将错误信息抛出，让全局监听的错误处理函数进行处理。

```
const convert = require("koa-convert");
const logger = require("./logger");

module.exports = (options) => {

  const loggerMiddleware = convert(logger(options));

  return (ctx, next) => {

    return loggerMiddleware(ctx, next)
    .catch((e) => {
        if (ctx.status < 500) {
            ctx.status = 500;
        }
        ctx.log.error(e.stack);
        ctx.state.logged = true;
        ctx.throw(e); 
    });
  };
}

```


## 总结

本节为大家介绍了写 log 中间件的具体步骤。然而作为例子还有很多不足，比如性能问题：写日志其实就是磁盘I/O过程，访问量大时，频繁写磁盘的操作会拖慢服务器；没有与监控系统紧密结合等。
log4js 中还有很多知识点，可以参考[官方文档](http://logging.apache.org/log4j/2.x/)，探索更多强大功能。
 
 









