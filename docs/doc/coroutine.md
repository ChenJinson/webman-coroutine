## 协程入门

本章节以swow的使用来举例介绍协程。

#### 1. 协程创建

Swow 的协程是面向对象的，所以我们可以这样创建一个待运行的协程
```php
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

```shell
/** @desc 任务1  */
function task1(): void
{
    // 总共3s
    for ($i = 0; $i < 3; $i++) {
        // 写入文件
        sleep(1);
        echo '[x] [🕷️] [写入文件] [' . $i . '] ' . date('Y-m-d H:i:s') . PHP_EOL;
    }
}

/** @desc 任务2 */
function task2(): void
{
    // 总共5s
    for ($i = 0; $i < 5; $i++) {
        // 发送邮件给50名会员,
        sleep(1);
        echo '[x] [🍁] [发送邮件] [' . $i . '] ' . date('Y-m-d H:i:s') . PHP_EOL;
    }
}

/** @desc 任务3  */
function task3(): void
{
    // 总共10s
    for ($i = 0; $i < 10; $i++) {
        // 模拟插入10
        sleep(1);
        echo '[x] [🌾] [插入数据] [' . $i . '] ' . date('Y-m-d H:i:s') . PHP_EOL;
    }
}
```
#### 普通请求执行
**执行代码**
```php
$timeOne = microtime(true);
task1();
task2();
task3();
$timeTwo = microtime(true);
echo '[x] [运行时间] ' . ($timeTwo - $timeOne) . PHP_EOL;
```
**打印结果**
```shell
[x] [🕷️] [写入文件] [0] 2024-09-28 08:54:26
[x] [🕷️] [写入文件] [1] 2024-09-28 08:54:27
[x] [🕷️] [写入文件] [2] 2024-09-28 08:54:28
[x] [🍁] [发送邮件] [0] 2024-09-28 08:54:29
[x] [🍁] [发送邮件] [1] 2024-09-28 08:54:30
[x] [🍁] [发送邮件] [2] 2024-09-28 08:54:31
[x] [🍁] [发送邮件] [3] 2024-09-28 08:54:32
[x] [🍁] [发送邮件] [4] 2024-09-28 08:54:33
[x] [🌾] [插入数据] [0] 2024-09-28 08:54:34
[x] [🌾] [插入数据] [1] 2024-09-28 08:54:35
[x] [🌾] [插入数据] [2] 2024-09-28 08:54:36
[x] [🌾] [插入数据] [3] 2024-09-28 08:54:37
[x] [🌾] [插入数据] [4] 2024-09-28 08:54:38
[x] [🌾] [插入数据] [5] 2024-09-28 08:54:39
[x] [🌾] [插入数据] [6] 2024-09-28 08:54:40
[x] [🌾] [插入数据] [7] 2024-09-28 08:54:41
[x] [🌾] [插入数据] [8] 2024-09-28 08:54:42
[x] [🌾] [插入数据] [9] 2024-09-28 08:54:43
[x] [运行时间] 18.004005908966
```

> 可以看出以上代码是`顺序执行`的，执行运行时间`18.004005908966`秒

#### 🚀 协程加持执行

**执行代码**
```php
$timeOne = microtime(true);
\Swow\Coroutine::run(function () {
    task1();
});
\Swow\Coroutine::run(function () {
    task2();
});
\Swow\Coroutine::run(function () {
    task3();
});
$timeTwo = microtime(true);
echo '[x] [运行时间] ' . ($timeTwo - $timeOne) . PHP_EOL;
```

**打印结果**
```shell
[x] [运行时间] 5.5074691772461E-5
```
> 这是因为协程化以后，协程之间是异步的，主协程并没有等待任务的协程结果，所以执行时间`5.5074691772461E-5`秒。

**改造代码**

- 使用waitGroup
```php
$wg = new \Swow\Sync\WaitGroup();
$wg->add(3);
$timeOne = microtime(true);
\Swow\Coroutine::run(function () use ($wg) {
    task1();
    $wg->done();
});
\Swow\Coroutine::run(function () use ($wg) {
    task2();
    $wg->done();
});
\Swow\Coroutine::run(function () use ($wg) {
    task3();
    $wg->done();
});
// 等待协程完毕
$wg->wait();
$timeTwo = microtime(true);
echo '[x] [运行时间] ' . ($timeTwo - $timeOne) . PHP_EOL;
```

- 使用waitAll()，**webman/workerman环境下请使用waitGroup**
```php
$timeOne = microtime(true);
\Swow\Coroutine::run(function () {
    task1();
});
\Swow\Coroutine::run(function () {
    task2();
});
\Swow\Coroutine::run(function () {
    task3();
});
// 等待协程完毕
waitAll();
$timeTwo = microtime(true);
echo '[x] [运行时间] ' . ($timeTwo - $timeOne) . PHP_EOL;
```

**打印结果**
```shell
[x] [🕷️] [写入文件] [0] 2024-09-28 09:02:46
[x] [🍁] [发送邮件] [0] 2024-09-28 09:02:46
[x] [🌾] [插入数据] [0] 2024-09-28 09:02:46
[x] [🕷️] [写入文件] [1] 2024-09-28 09:02:47
[x] [🍁] [发送邮件] [1] 2024-09-28 09:02:47
[x] [🌾] [插入数据] [1] 2024-09-28 09:02:47
[x] [🕷️] [写入文件] [2] 2024-09-28 09:02:48
[x] [🍁] [发送邮件] [2] 2024-09-28 09:02:48
[x] [🌾] [插入数据] [2] 2024-09-28 09:02:48
[x] [🍁] [发送邮件] [3] 2024-09-28 09:02:49
[x] [🌾] [插入数据] [3] 2024-09-28 09:02:49
[x] [🍁] [发送邮件] [4] 2024-09-28 09:02:50
[x] [🌾] [插入数据] [4] 2024-09-28 09:02:50
[x] [🌾] [插入数据] [5] 2024-09-28 09:02:51
[x] [🌾] [插入数据] [6] 2024-09-28 09:02:52
[x] [🌾] [插入数据] [7] 2024-09-28 09:02:53
[x] [🌾] [插入数据] [8] 2024-09-28 09:02:54
[x] [🌾] [插入数据] [9] 2024-09-28 09:02:55
[x] [运行时间] 9.4166378974915
```

> 主协程等待子协程，子协程交替运行，执行时间`9.4166378974915`秒。
