这是一次分享准备

自己还没有总结这个的能力，这次就当个搬运工好了

## 工具准备
PHPSTORM,不只是编程。

个人觉得只要能股提供全局搜索、单页搜索、函数跳转的编辑器就能够满足要求。一般在审计的时候，先找到一个入口点，配合代码和浏览器，一边下断电，一边打变量，再在浏览器上观察反应。

自动化审计工具，食之无味弃之可惜。

当然，我手边能用到的自动化审计工具就是能在网上找到资源的那几种。对于快速定位漏洞点来审计，自动化工具有一定的作用的。但是，大多数我能接触到的审计工具大抵是基于规则匹配的或者有一定的数据流向，但由于不能做到语义识别，很多逻辑控制语句无法识别，导致敏感点可控参数无法自由的跳转组合。

现在的习惯是，有了代码，先会扔进审计工具跑一跑，看一个大概趋势，如果时间充裕，一般还是愿意一点一点从代码入口点跟进。

## 审计初步
审计可以从多个角度切入，首先是搞清楚审计对象的架构，是套了开源的框架，还是原生代码，MVC，还是仅仅是函数调用。审计的流程一般是从软件的index入口点开始，逐渐遍历到所有函数文件。

那么在审计之前，最好就是使用或者熟悉这些常见框架的特性机制。
- CI https://codeigniter.org.cn/
- YII2 http://www.yiichina.com/
- ThinkPHP http://www.thinkphp.cn/
- Laravel https://laravel.com/
熟悉这些框架方便我们快速判断审计点。比如之前的一次代码审计就是套了ThinkPHP的核心代码，结果导致ThinkPHP的漏洞对于那套系统依然有效

为了能够边学习边实践别人的思想，我这里以MetInfo为例进行说明。

对于快速审计，比如自动化审计工具，往往可以考虑以下一些切入点。
### 敏感函数切入
敏感函数包括能够代码执行(PHP、MYSQL)的函数、上传下载文件的函数、文件读取函数、加载资源函数。
- eval() 海洋CMS6.28
    `@eval("if(".$strIf."){\$ifFlag=true;}else{\$ifFlag=false;}");`
- exec() imo 云办公
    `$result = exec($_POST['command']);`
- preg_replace() Thinkphp2.1
    ```PHP
    preg_replace('/test/e', 'phpinfo()', 'just test');
    preg_replace('@(w+)'.$depr.'([^'.$depr.'\/]+)@e', '$var[\'\1\']="\2";', implode($depr,$paths));
    ```
- system() Family Connections CMS v2.5.0-v2.7.1
- assert()
- call_user_func()
- call_user_func_array()
- create_function()
- shell_exec()
- passthru() -Narcissus在线图像汇编器
- escapeshellcmd()
- pcntl_exec()
- 反括号 大括号{}
- ob_start()
- array_map()
- proc_open()
- popen()
- require_once()
- include_once()
- file_get_contents()
- highlight_file()
- fopen()
- readfile()
- fread()
- fgetss()
- fgets()
- parse_ini_file()
- show_source()
- file()
- echo XSS
……

这一类的函数有很多，这种漏洞一般可能会呈现两种趋势，一种就是常用函数过滤绕过，一种就是不常见函数用法失误。

### 漏洞点切入
如果说白盒找敏感函数的话，黑盒上对应的就是优先查找程序的QSL执行点、上传点、登陆点、密码找回点等
- SQL注入
登录页面、表单接受页面、信息显示页面

普通拼接这样不多说了，主流开源的代码想挖到这种已经很难了，要么没有拼接的用法，那么就做了更加严格的过滤。

当然，这里有些小技巧，比如多次过滤情况下的绕过。如`include\mail\class.phpmailer.php`的1741行：
```PHP
$textMsg = trim(strip_tags(preg_replace('/<(head|title|style|script)[^>]*>.*?<\/\\1>/s','',$message)));
```
连续使用了两个过滤函数，这两个过滤函数在一起就容易造成绕过。可以使用这两个payload做对比：
```HTML
<head>evil</head>
<he<>ad>evil</head>
```
- 编码注入
addslashes、mysql_real_escape_string、mysql_escape_string或者开启GPC来转义，但是，如果再次使用urlencode就会出现二次编码。
- 文件包含
模块加载，模板加载，cache调用
- 任意文件读取
寻找文件读取敏感函数
- 文件上传
move_uploaded_file()
- 文件删除
unlink()
- 变量覆盖
`extract()`、`parse_str()`、`$$`

## 审计实践
这里以Metinfo为了进行演示，先大致了解一下这款CMS的结构。

根目录下入口index.php，前台客户逻辑在`/member/`里，后台逻辑在`/admin/`里，自己创建的应用在`/app/`中，前台的下载/搜索等逻辑对应在相应的目录下。

常用的功能函数集中在`/include/`下，重点在`*.inc.php`和`*.func.php`中。
```PHP
/index.php
    |
/include/common.inc.php
```
这个文件很有意思，比如里面有名的变量覆盖，由变量覆盖处跳转到过滤函数`daddlashes()`，明显这有打补丁的痕迹，分享时现场细说，这里就不赘述。
```PHP
/include/common.inc.php
    |
/include/head.php
    |
/app/
    |
/include/
```
MetInfo由于特殊的变量传递机制和路由机制，使得我们可以轻易访问任意PHP文件并且携带参数进行测试，所以对于它的审计顺序会更加松散一点。

我这里先从关联性较弱的`/about/`、`/upload/`、`/search/`等几个文件夹看起，最后再集中阅读用于应用的`/app/`文件夹。

利用前面说的断电输出变量的方法一点一点调试实例，分享用`http://127.0.0.1/case/index.php?metid=1&filpy=&fmodule=0&modulefname=2`。到这里都没有什么太大的发现，唯一的感受就是有很多点都很惊险，由于没有用MVC框架，显然很多SQL语句之前写法差异性都比较大。

之后就可以跟进`include`的功能函数来看，也就是在这里终于知道了一个小漏洞。

前台大概看得差不多了，就可以跟进后台admin文件夹看看，这里就会觉得轻松很多，一个后台就不再适用过滤函数了，另一个后台功能更多一点，不过后台漏洞大多数智能算代码BUG，没有前台来得那么惊艳。

大多数的内容打算分享的时候直接口述，这里其实只能算作一个大纲，所以文字上不是很通顺。
