# webman-coroutine

## 🐞 简介

> **🚀🚀 webman-coroutine 是一个 webman 开发框架生态下的协程基建支撑插件**

**主要实现以下功能**：

1. 支持`workerman 4.x`的 swow 协程驱动能力，兼容`workerman 5.x`版本自带的`swow`协程驱动；
2. 支持`workerman 4.x`的 swoole 协程驱动能力，兼容`workerman 5.x`版本自带的`swoole`协程驱动；
3. 实现`coroutine web server` 用于实现具备协程能力的 web 框架基建
4. 支持自定义协程实现，如基于`revolt`等

## 🕷️ 说明

1. `workerman 4.x/5.x`驱动下的 webman 框架无法完整使用`swoole`的协程能力，所以使用`CoroutineWebServer`来替代`webman`自带的`webServer`
2. `workerman 4.x`下还未有官方支持的`swow`协程驱动，本插件提供`SwowEvent`事件驱动支撑`workerman 4.x`下的协程能力
3. 由于配置`event-loop`等操作相较于普通开发会存在一定的心智负担，所以本插件提供了`event_loop()`函数，用于根据当前环境自动选择合适的事件驱动

## 🪰 安装

通过`composer`安装

```php
composer require workbunny/webman-coroutine
```
> 注: 目前在开发阶段，体验请使用`dev-main`分支

**配置说明**

- enable : (true/false), 是否启用协程webServer
- port : (int), 协程webServer默认端口
- channel_size : (int), 每个connection的channel容量
- consumer_count : (int), 每个connection的消费者数量

## 📖 文档

[API文档](docs%2Findex.html)

## 🐜 教程

#### 1. swow 环境

1. 使用`./vendor/bin/swow-builder`安装`swow`拓展，注意请关闭`swoole`环境
2. 修改`config/server.php`中`'event_loop' => \Workbunny\WebmanCoroutine\event_loop()`，
   `event_loop()`函数会根据当前环境自行判断当前的 workerman 版本，自动选择合适的事件驱动
   - 当开启`swow`拓展时，`workerman 4.x`下使用`SwowEvent`事件驱动
   - 当开启`swow`拓展时，`workerman 5.x`下使用`workerman`自带的`Swow`事件驱动
   - 当未开启`swow`时，使用`workerman`自带的`Event`事件驱动
3. 使用`php -d extension=swow webman start`启动
4. webman 自带的 webServer 协程化，可以关闭启动的`CoroutineWebServer`

> 注：`CoroutineWebServer`可以在`config/plugin/workbunny/webman-coroutine/app.php`中通过`enable=false`关闭启动

#### 2. swoole 环境

1. 使用`pecl install swoole`安装稳定版 swoole 拓展
2. 建议不要将`swoole`加入`php.ini`配置文件
3. 修改`config/server.php`中`'event_loop' => \Workbunny\WebmanCoroutine\event_loop()`，
   `event_loop()`函数会根据当前环境自行判断当前的 workerman 版本，自动选择合适的事件驱动
   - 当开启 swoole 拓展时，workerman 4.x 下使用 SwooleEvent 事件驱动
   - 当开启 swoole 拓展时，workerman 5.x 下使用 workerman 自带的 Swoole 事件驱动
   - 当未开启 swoole 时，使用 workerman 自带的 Event 事件驱动
4. 使用`php -d extension=swoole webman start`启动
5. 通过`config/plugin/workbunny/webman-coroutine/process.php`启动的 CoroutineWebServer 可以用于协程环境开发，原服务还是 BIO 模式

#### 3. ripple 环境

1. 使用`composer require cclilshy/p-ripple-drive`安装 ripple 驱动插件
2. 修改`config/server.php`配置
   - `'event_loop' => \Workbunny\WebmanCoroutine\event_loop()`自动判断，请勿开启 swow、swoole，
   - `'event_loop' => \Workbunny\WebmanCoroutine\Factory::RIPPLE_FIBER`手动指定
3. 使用`php webman start`启动

> 注：该环境协程依赖`php-fiber`，并没有自动`hook`系统的阻塞函数，但支持所有支持`php-fiber`的插件

#### 4. 自定义环境

1. 实现`Workbunny\WebmanCoroutine\Handlers\HandlerInterface`接口，实现自定义协程处理逻辑
2. 通过`Workbunny\WebmanCoroutine\Factory::register(HandlerInterface $handler)`注册你的协程处理器
3. 修改`config/server.php`中`'event_loop' => {你的事件循环类}`
4. 启动`CoroutineWebServer` 接受处理协程请求

> 注：`\Workbunny\WebmanCoroutine\event_loop()`自动判断加载顺序按`\Workbunny\WebmanCoroutine\Factory::$_handlers`的顺序执行`available()`择先

> 注：因为`eventLoopClass`与`HandlerClass`是一一对应的，所以建议不管是否存在相同的事件循环或者相同的处理器都需要继承后重命名

## 自定义协程化

`webman-coroutine`提供了用于让自己的自定义服务/进程协程化的基础工具

> 注：考虑到 webman 框架默认不会启用注解代理，所以这里没有使用注解代理来处理协程化代理

#### 1. 自定义进程

假设我们已经存在一个自定义服务类，如`MyProcess.php`

```php
namespace process;

class MyProcess {
    public function onWorkerStart() {
        // 具体业务逻辑
    }
    // ...
}
```

在`webman/workerman`环境中，`onWorkerStart()`是一个 worker 进程所必不可少的方法，
假设我们想要将它协程化，在不改动`MyProcess`的情况下，只需要新建一个`MyCoroutineProcess.php`

```php
namespace process;

use Workbunny\WebmanCoroutine\CoroutineWorkerInterface;
use Workbunny\WebmanCoroutine\CoroutineWorkerMethods;

class MyCoroutineProcess extends MyProcess implements CoroutineWorkerInterface {

    // 引入协程代理方法
    use CoroutineWorkerMethods;
}
```

此时的`MyCoroutineProcess`将拥有协程化的`onWorkerStart()`，将新建的`MyCoroutineProcess`添加到 webman 的自定义进程配置`config/process.php`中启动即可

#### 2. 自定义服务

> 代码样例：[CoroutineWebServer.php](src%2FCoroutineWebServer.php)

假设我们已经存在一个自定义服务类，如`MyServer.php`

```php
namespace process;

class MyServer {

    public function onMessage($connection, $data) {
        // 具体业务逻辑
    }

    // ...
}
```

在`webman/workerman`环境中，`onMessage()`是一个具备监听能力的进程所必不可少的方法，假设我们想要将它协程化，在不改动`MyServer`的情况下，只需要新建一个`MyCoroutineServer.php`

```php
namespace process;

use Workbunny\WebmanCoroutine\CoroutineServerInterface;
use Workbunny\WebmanCoroutine\CoroutineServerMethods;

class MyCoroutineServer extends MyServer implements CoroutineServerInterface {

    // 引入协程代理方法
    use CoroutineServerMethods;
}
```

此时的`MyCoroutineServer`将拥有协程化的`onMessage()`，将新建的`MyCoroutineServer`添加到 webman 的自定义进程配置`config/process.php`中启动即可

## 协程入门

#### 1. 协程创建

Swow 的协程是面向对象的，所以我们可以这样创建一个待运行的协程
```
use Swow\Coroutine;

$coroutine = new Coroutine(static function (): void {
    echo "Hello 开源技术小栈\n";
});
```
这样创建出来的协程并不会被运行，而是只进行了内存的申请。

#### 2. 协程的观测

通过 `var_dump` 打印协程对象，我们又可以看到这样的输出：
```
var_dump($coroutine);
```
打印输出
```ts
class Swow\Coroutine#240 (4) {
  public $id =>
  int(12)
  public $state =>
  string(7) "waiting"
  public $switches =>
  int(0)
  public $elapsed =>
  string(3) "0ms"
}
```
从输出我们可以得到一些协程状态的信息，如：协程的 `id` 是`12`，状态是`等待中`，切换次数是`0`，运行了`0`毫秒（即没有运行）。

通过 `resume()` 方法，我们可以唤醒这个协程：
```
$coroutine->resume();
```
协程中的PHP代码被执行，于是我们就看到了下述信息：
```yaml
Hello 开源技术小栈
```
这时候我们再通过 `var_dump($coroutine);` 去打印协程的状态，我们得到以下内容：
```ts
class Swow\Coroutine#240 (4) {
  public $id =>
  int(12)
  public $state =>
  string(4) "dead"
  public $switches =>
  int(1)
  public $elapsed =>
  string(3) "0ms"
}
```
可以看到协程已经运行完了所有的代码并进入`dead`状态，共经历一次协程切换。

## 协程实战

#### 多进程和协程执行顺序

![image](https://github.com/user-attachments/assets/16fb3138-52ae-4ed1-9c15-bf51c6151fe3)

#### 实战伪代码

```ts
/** @desc 任务1  */
function task1(): void
{
    for ($i = 0; $i <= 50; $i++) {
        // 写入文件,大概要3000微秒
        usleep(3000);
        echo '[x] [🕷️] [写入文件] [' . $i . '] ' . date('Y-m-d H:i:s') . PHP_EOL;
    }
}

/** @desc 任务2 */
function task2(): void
{
    for ($i = 0; $i <= 100; $i++) {
        // 发送邮件给100名会员,大概3000微秒
        usleep(3000);
        echo '[x] [🍁] [发送邮件] [' . $i . '] ' . date('Y-m-d H:i:s') . PHP_EOL;
    }
}

/** @desc 任务3  */
function task3(): void
{
    for ($i = 0; $i <= 150; $i++) {
        // 模拟插入150条数据,大概3000微秒
        usleep(3000);
        echo '[x] [🌾] [插入数据] [' . $i . '] ' . date('Y-m-d H:i:s') . PHP_EOL;
    }
}
```
#### 普通请求执行
**执行代码**
```
$timeOne = microtime(true);
task1();
task2();
task3();
$timeTwo = microtime(true);
echo '[x] [运行时间] ' . ($timeTwo - $timeOne) . PHP_EOL;
```
**打印结果**
```ts
[x] [运行开始时间] 1727454935.2908
[x] [🕷️] [写入文件] [0] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [1] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [2] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [3] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [4] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [5] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [6] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [7] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [8] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [9] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [10] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [11] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [12] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [13] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [14] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [15] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [16] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [17] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [18] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [19] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [20] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [21] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [22] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [23] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [24] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [25] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [26] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [27] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [28] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [29] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [30] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [31] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [32] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [33] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [34] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [35] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [36] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [37] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [38] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [39] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [40] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [41] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [42] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [43] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [44] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [45] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [46] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [47] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [48] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [49] 2024-09-28 00:21:48
[x] [🕷️] [写入文件] [50] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [0] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [1] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [2] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [3] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [4] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [5] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [6] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [7] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [8] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [9] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [10] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [11] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [12] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [13] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [14] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [15] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [16] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [17] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [18] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [19] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [20] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [21] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [22] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [23] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [24] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [25] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [26] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [27] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [28] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [29] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [30] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [31] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [32] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [33] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [34] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [35] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [36] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [37] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [38] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [39] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [40] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [41] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [42] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [43] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [44] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [45] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [46] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [47] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [48] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [49] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [50] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [51] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [52] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [53] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [54] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [55] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [56] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [57] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [58] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [59] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [60] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [61] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [62] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [63] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [64] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [65] 2024-09-28 00:21:48
[x] [🍁] [发送邮件] [66] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [67] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [68] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [69] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [70] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [71] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [72] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [73] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [74] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [75] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [76] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [77] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [78] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [79] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [80] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [81] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [82] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [83] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [84] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [85] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [86] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [87] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [88] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [89] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [90] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [91] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [92] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [93] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [94] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [95] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [96] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [97] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [98] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [99] 2024-09-28 00:21:49
[x] [🍁] [发送邮件] [100] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [0] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [1] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [2] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [3] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [4] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [5] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [6] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [7] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [8] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [9] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [10] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [11] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [12] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [13] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [14] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [15] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [16] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [17] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [18] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [19] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [20] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [21] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [22] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [23] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [24] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [25] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [26] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [27] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [28] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [29] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [30] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [31] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [32] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [33] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [34] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [35] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [36] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [37] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [38] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [39] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [40] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [41] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [42] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [43] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [44] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [45] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [46] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [47] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [48] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [49] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [50] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [51] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [52] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [53] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [54] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [55] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [56] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [57] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [58] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [59] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [60] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [61] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [62] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [63] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [64] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [65] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [66] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [67] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [68] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [69] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [70] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [71] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [72] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [73] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [74] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [75] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [76] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [77] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [78] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [79] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [80] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [81] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [82] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [83] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [84] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [85] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [86] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [87] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [88] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [89] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [90] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [91] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [92] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [93] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [94] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [95] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [96] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [97] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [98] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [99] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [100] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [101] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [102] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [103] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [104] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [105] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [106] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [107] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [108] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [109] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [110] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [111] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [112] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [113] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [114] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [115] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [116] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [117] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [118] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [119] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [120] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [121] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [122] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [123] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [124] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [125] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [126] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [127] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [128] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [129] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [130] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [131] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [132] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [133] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [134] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [135] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [136] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [137] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [138] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [139] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [140] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [141] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [142] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [143] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [144] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [145] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [146] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [147] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [148] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [149] 2024-09-28 00:21:49
[x] [🌾] [插入数据] [150] 2024-09-28 00:21:49
[x] [运行时间] 0.93667697906494
```

> 可以看出以上代码是`顺序执行`的，执行运行时间`0.9336`秒
```ts
[x] [运行开始时间] 1727454935.2908
[x] [运行结束时间] 1727454936.2245
```
#### 🚀 协程加持执行
**执行代码**
```
$timeOne = microtime(true);
echo '[x] [运行开始时间] ' . $timeOne . PHP_EOL;

/** 协程1 */
$coroutine1 = new \Swow\Coroutine(static function (): void {
    task1();
});
$coroutine1->resume();

/** 协程2 */
$coroutine2 = new \Swow\Coroutine(static function (): void {
    task2();
});
$coroutine2->resume();

/** 协程3 */
$coroutine3 = new \Swow\Coroutine(static function (): void {
    task3();
});
$coroutine3->resume();
$timeTwo = microtime(true);
echo '[x] [运行结束时间] ' . $timeTwo . PHP_EOL;
```
**打印结果**
```ts
[x] [运行开始时间] 1727454795.2326
[x] [运行结束时间] 1727454795.2328
[x] [🕷️] [写入文件] [0] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [0] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [0] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [1] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [1] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [1] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [2] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [2] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [2] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [3] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [3] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [3] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [4] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [4] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [4] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [5] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [5] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [5] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [6] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [6] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [6] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [7] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [7] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [7] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [8] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [8] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [8] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [9] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [9] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [9] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [10] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [10] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [10] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [11] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [11] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [11] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [12] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [12] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [12] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [13] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [13] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [13] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [14] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [14] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [14] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [15] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [15] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [15] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [16] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [16] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [16] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [17] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [17] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [17] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [18] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [18] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [18] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [19] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [19] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [19] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [20] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [20] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [20] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [21] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [21] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [21] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [22] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [22] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [22] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [23] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [23] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [23] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [24] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [24] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [24] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [25] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [25] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [25] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [26] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [26] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [26] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [27] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [27] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [27] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [28] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [28] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [28] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [29] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [29] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [29] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [30] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [30] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [30] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [31] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [31] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [31] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [32] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [32] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [32] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [33] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [33] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [33] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [34] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [34] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [34] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [35] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [35] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [35] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [36] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [36] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [36] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [37] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [37] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [37] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [38] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [38] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [38] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [39] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [39] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [39] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [40] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [40] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [40] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [41] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [41] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [41] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [42] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [42] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [42] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [43] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [43] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [43] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [44] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [44] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [44] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [45] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [45] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [45] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [46] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [46] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [46] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [47] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [47] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [47] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [48] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [48] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [48] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [49] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [49] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [49] 2024-09-28 00:33:15
[x] [🕷️] [写入文件] [50] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [50] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [50] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [51] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [51] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [52] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [52] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [53] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [53] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [54] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [54] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [55] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [55] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [56] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [56] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [57] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [57] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [58] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [58] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [59] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [59] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [60] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [60] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [61] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [61] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [62] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [62] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [63] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [63] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [64] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [64] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [65] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [65] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [66] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [66] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [67] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [67] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [68] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [68] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [69] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [69] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [70] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [70] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [71] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [71] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [72] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [72] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [73] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [73] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [74] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [74] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [75] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [75] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [76] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [76] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [77] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [77] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [78] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [78] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [79] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [79] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [80] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [80] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [81] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [81] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [82] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [82] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [83] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [83] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [84] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [84] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [85] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [85] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [86] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [86] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [87] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [87] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [88] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [88] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [89] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [89] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [90] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [90] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [91] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [91] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [92] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [92] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [93] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [93] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [94] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [94] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [95] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [95] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [96] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [96] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [97] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [97] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [98] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [98] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [99] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [99] 2024-09-28 00:33:15
[x] [🍁] [发送邮件] [100] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [100] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [101] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [102] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [103] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [104] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [105] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [106] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [107] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [108] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [109] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [110] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [111] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [112] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [113] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [114] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [115] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [116] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [117] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [118] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [119] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [120] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [121] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [122] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [123] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [124] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [125] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [126] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [127] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [128] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [129] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [130] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [131] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [132] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [133] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [134] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [135] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [136] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [137] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [138] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [139] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [140] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [141] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [142] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [143] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [144] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [145] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [146] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [147] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [148] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [149] 2024-09-28 00:33:15
[x] [🌾] [插入数据] [150] 2024-09-28 00:33:15
```
> 可以看出以上代码是`交叉执行`。**执行运行时间只需要`0.0002`秒**，而且接口请求没有任何阻塞。
```lua
[x] [运行开始时间] 1727454795.2326
[x] [运行结束时间] 1727454795.2328
```
## ♨️ 相关文章

* [webman如何使用swow事件驱动和协程？](https://mp.weixin.qq.com/s?__biz=MzUzMDMxNTQ4Nw==&mid=2247496493&idx=1&sn=4ab95befc894d556eac26d405f354a40&chksm=fa51129dcd269b8b61fc5b1a15a9a23b99b61c0780b9a341dfe3733692e85a1bc5e323ee9775#rd)
* [PHP高性能纯协程网络通信引擎Swow](https://mp.weixin.qq.com/s?__biz=MzUzMDMxNTQ4Nw==&mid=2247496428&idx=1&sn=5f1fef3a49e3ab20ea1fa43242ac8af7&chksm=fa51135ccd269a4aac1255323faeea670238777c37fec6fb6bdef0ead857ba492c1265c03bff#rd)
* [workerman5.0 和 swoole5.0 实现一键协程](https://mp.weixin.qq.com/s?__biz=MzUzMDMxNTQ4Nw==&mid=2247492324&idx=1&sn=ac697103fe56d6054593ae6d1bdadb93&chksm=fa510354cd268a4298eee50483821fff3ebb52a923a6a67708759ea4c5836649c85700f9ad12#rd)
* [webman如何使用swoole事件驱动和协程？](https://mp.weixin.qq.com/s?__biz=MzUzMDMxNTQ4Nw==&mid=2247489841&idx=1&sn=52e9a57e511870c68daa2b10b78bf3a2&chksm=fa52f881cd25719782e3162108426a127b80599df80633d5edcf164162a69dc3518a9ec9cd29#rd)

## 💕 致谢
>> **💕感恩 workerman 和 swow 开发团队为 PHP 社区带来的创新和卓越贡献，让我们共同期待 PHP 在实时应用领域的更多突破！！！**
