# swoole协程

### 创建协程

```php
    echo "test-1\n";
    go(function(){
        Co::sleep(2);  // 协程让出控制，进入协程控制队列。继续往下执行
        echo "coroutine 111 \n";
    });

    echo "test-2";
    go(function(){
        echo "coroutine 222 \n";
    });

    echo "test-3\n";
```
<!--more-->
![Image text](/images/go1.jpg)
- go() 是 \Co::create() 的缩写, 用来创建一个协程
- 执行过程:
    >>1、运行php Demo.php 系统启动一个新进程
    >>2、执行到go()函数，在当前进程中生成一个协程
    >>3、协程中遇到IO阻塞(Co::sleep(2)),协程让出控制, 进入协程调度队列
    >>4、进程继续向下执行，输出：test-2 
    >>5、执行下一个协程, 输出：coroutine 222
    >>6、之前的协程准备就绪, 继续执行, 输出：coroutine 111
- 结果：通过以上代码例子，可以了解到协程和进程的关系，协程的调度

### 协程为什么快，快在哪

- 常见说法： 可以创建很多个协程来执行任务, 所以快
- 计算机任务分为2种
    > 1、CPU密集型： 比如加减乘除等科学计算
    > 2、IO 密集型： 比如网络请求, 文件读写等
- 并发
  > 由于CPU切换任务非常快, 快到人类可以感知的极限, 就会有很多任务同时执行的错觉
- 并行
  > 同一个时刻, 同一个CPU只能执行同一个任务, 要同时执行多个任务, 就需要有多个CPU才行
- 协程适合的是 IO 密集型应用, 因为协程在 IO阻塞时会自动调度, 减少IO阻塞导致的时间损失


普通程序运行如下：
```php
$n = 3;
for($i=0; $i < $n; $i++){
    sleep(1);
    echo microtime(true) . " : hello {$i}".PHP_EOL;
}
echo "hello main".PHP_EOL;
```
![Image text](/images/test1.jpg)

多协程运行如下：
```php
$n = 3;
for ($i = 0; $i < $n; $i++) {
    go(function () use ($i) {
        Co::sleep(1);
        echo microtime(true) . ": hello {$i}".PHP_EOL;
    });
};
echo "hello main".PHP_EOL;
```
![Image text](/images/test2.jpg)
- 结果
  > 1、普通的程序运行时遇到 IO阻塞 会导致的性能损失
  > 2、多协程，遇到IO阻塞时发生调度, IO就绪时恢复运行
- 说明
  > 1、sleep() 可以看做是 CPU密集型任务, 不会引起协程的调度
  > 2、Co::sleep() 模拟的是 IO密集型任务, 会引发协程的调度
### 管道（channel）

- Channel 管道：支持多生产者协程和多消费者协程。底层自动实现了协程的切换和调度。
- Channel 管道是用于同一进程内协程之间交换数据的工具，可以理解为，Channel 就是一个实现了协程切换和调度的队列，亦或是数组。
- 生产协程:在channel已满时，会被挂起；
- 消费协程:在channel为空是，也会被挂起。

```php
$chan = new Chan();
go(function()use($chan){
        for($i=0;$i<5;$i++){
                $chan->push($i);
                echo "顺序插入{$i}".PHP_EOL;
        }
});
echo "顺序执行".PHP_EOL;
go(function()use($chan){
        while(!$chan->isEmpty()){
                $res = $chan->pop();
                echo "顺序消费{$res}".PHP_EOL;
        }
});

```
![Image text](/images/chan.jpg)
> 结果可以看出生产者协程和消费者协程是交替运行的，而协程切换的时机则是在运行到 push 和 pop 的时候，首先会进入生产者协程，然后生产了一条数据，然后代码继续执行输出“顺序执行”的字符串并创建了消费者协程；由于前面已经 push 了一条数据所以此时的 $channel->isEmpty() 是非空状态，再执行 pop。

### 连接池

- 由于管道(channel)的特性（写入消费），可以通过管道实现连接池
- 连接池是一个用于分配和管理连接的容器，可以避免在高并发的系统下反复地去创建和销毁连接，便于连接的复用。

```php
<?php 

class Mysql
{
    public $pool;
    public $config = [
        'maxnum' => 20, // 最大连接数
        'mysql' => [  // 数据库配置
            'host'=>'192.168.1.13',
            'port'=>3306,
            'user'=>'root',
            'password'=>'123456',
            'database'=>'test'
        ]
    ];
    /** 初始化 */
    public function __construct()
    {
        $maxnum = $this->config['maxnum'];
        $this->pool = new \Swoole\Coroutine\Channel($maxnum);
        for($i=1; $i<$maxnum; $i++){
            $mysqlConnect = $this->createConnect();
            $this->push($mysqlConnect);
        }
    }
    /** 创建数据库连接 */
    public function createConnect()
    {
        $mysql = new \Swoole\Coroutine\MySQL();  // 使用连接mysql组件
        $mysql->connect($this->config['mysql']);  // 连接操作
        return $mysql;
    }
    /** 生产者插入 */
    public function push($source){
        return $this->pool->push($source);
    }
    /** 销毁 */
    public function pop(){
        return $this->pool->pop();
    }
    /** 当前连接数 */
    public function length(){
        return $this->pool->length();
    }
}
// 创建一个协程
go(function(){
    $model = new Mysql();
    $conn = $model->pop(); //  获取连接
    $res = $conn->query('show tables');  // 查询当前库的有哪些表
    // 逐条打印表名
    foreach($res as $key => $val){
        echo $val['Tables_in_test'].PHP_EOL;
    }
});
```
![Image text](/images/mysql.jpg)

