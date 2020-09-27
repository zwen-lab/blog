### 问题说明

laravel 版本5.2.45 ，队列驱动为数据库时，laravel没有对连接断开的情况进行处理，如果连接端口会导致队列不可用。



### 重现步骤

##### 1.建表

```
php artisan queue:table
php artisan migrate
```

##### 2.队列驱动配置成数据库 

```php
#.env文件数据库驱动配置成database
QUEUE_DRIVER=database
```

##### 3.启动队列

```
php artisan queue:work  --daemon --sleep=3 --tries=3  
```

##### 4.模拟连接断开

在mysql命令行通过show processlist 命令找到自己队列的连接并杀掉这个连接，模拟出连接超时断开的情况。

##### 5.错误信息

```
[2020-09-27 10:43:34] local.ERROR: PDOException: SQLSTATE[HY000]: General error: 2006 MySQL server has gone away in /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Database/Connection.php:576
Stack trace:
#0 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Database/Connection.php(576): PDO->beginTransaction()
#1 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Queue/DatabaseQueue.php(162): Illuminate\Database\Connection->beginTransaction()
#2 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Queue/Worker.php(180): Illuminate\Queue\DatabaseQueue->pop()
#3 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Queue/Worker.php(149): Illuminate\Queue\Worker->getNextJob()
#4 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Queue/Worker.php(111): Illuminate\Queue\Worker->pop()
#5 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Queue/Worker.php(86): Illuminate\Queue\Worker->runNextJobForDaemon()
#6 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Queue/Console/WorkCommand.php(119): Illuminate\Queue\Worker->daemon()
#7 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Queue/Console/WorkCommand.php(78): Illuminate\Queue\Console\WorkCommand->runWorker()
#8 [internal function]: Illuminate\Queue\Console\WorkCommand->fire()
#9 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Container/Container.php(507): call_user_func_array()
#10 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Console/Command.php(169): Illuminate\Container\Container->call()
#11 /data/wwwroot/new.new.console.admin/vendor/symfony/console/Command/Command.php(256): Illuminate\Console\Command->execute()
#12 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Console/Command.php(155): Symfony\Component\Console\Command\Command->run()
#13 /data/wwwroot/new.new.console.admin/vendor/symfony/console/Application.php(794): Illuminate\Console\Command->run()
#14 /data/wwwroot/new.new.console.admin/vendor/symfony/console/Application.php(186): Symfony\Component\Console\Application->doRunCommand()
#15 /data/wwwroot/new.new.console.admin/vendor/symfony/console/Application.php(117): Symfony\Component\Console\Application->doRun()
#16 /data/wwwroot/new.new.console.admin/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(107): Symfony\Component\Console\Application->run()
#17 /data/wwwroot/new.new.console.admin/artisan(35): Illuminate\Foundation\Console\Kernel->handle()
#18 {main}  
```

### 原因分析

```php
   //下面的代码去掉了许多其他代码，只保留了触发报错的代码 
   
    /**
     * laravel\framework\src\Illuminate\Queue\DatabaseQueue.php 
     * Pop the next job off of the queue.
     * 
     * @param  string  $queue
     * @return \Illuminate\Contracts\Queue\Job|null
     */
    public function pop($queue = null)
    {  
        $this->database->beginTransaction();  //开启一个事物，也就是这里开始触发问题
    }

    /**
     * laravel\framework\src\Illuminate\Database\Connection.php
     * Start a new database transaction.
     *
     * @return void
     * @throws Exception
     */
    public function beginTransaction()
    {
         $this->getPdo()->beginTransaction(); //先获取pdo实例，然后开启一个事物
    }
   
    /**
     * laravel\framework\src\Illuminate\Database\Connection.php
     * Get the current PDO connection.
     *
     * @return \PDO
     */
    public function getPdo()
    {
        //第一次调用的时候是Closure 所以调用Closure 获得\PDO实例连接，之后就直接读取这个实例进连接
        //当这个连接被异常断开的时候，问题就产生了，没有进行连接断开的异常处理
        if ($this->pdo instanceof Closure) { 
            return $this->pdo = call_user_func($this->pdo);
        }

        return $this->pdo;
    }
     
```

### 解决方案

##### 1.直接修改框架源码（如果你的项目中早已经改过了，就接着破罐子破摔吧）

修改 laravel\framework\src\Illuminate\Queue\DatabaseQueue.php 文件

```php
//引入 use Illuminate\Database\DetectsLostConnections 这个trait

/**
     * Pop the next job off of the queue.
     *
     * @param  string  $queue
     * @return \Illuminate\Contracts\Queue\Job|null
     */
    public function pop($queue = null)
    {
        $queue = $this->getQueue($queue);

        //添加异常处理，如果是连接断开，进行一次重连
        try{
            $this->database->beginTransaction(); 
        }catch(\Exception $e){
            if($this->causedByLostConnection($e)){
                $this->database->reconnect();
                $this->database->beginTransaction();
            }else{
                throw $t;
            }
        }
       ...
    }
```



##### 2.不直接修改框架底层源码

1.拷贝laravel\framework\src\Illuminate\Queue\DatabaseQueue.php  到 vendor-overwrite/DatabaseQueue.php，然后pop相关的代码修改成上面的代码。

2.项目的conposer.json 文件的autoload -> classmap 下面添加如下配置

```
"vendor-overwrite/DatabaseQueue.php",
```

3.执行comoser dump-autoload 命令



###### 3.队列不采用database 驱动，改用redis等驱动，经过测试redis不存在这个情况。

