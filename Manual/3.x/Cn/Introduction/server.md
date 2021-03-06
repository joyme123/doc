# 服务管理脚本
执行完框架安装后，可以在你的项目根目录下，看多一个easyswoole的文件。
执行以下命令：
```
php easyswoole
```
可见：
```
 ______                          _____                              _
 |  ____|                        / ____|                            | |
 | |__      __ _   ___   _   _  | (___   __      __   ___     ___   | |   ___
 |  __|    / _` | / __| | | | |  \___ \  \ \ /\ / /  / _ \   / _ \  | |  / _ \
 | |____  | (_| | \__ \ | |_| |  ____) |  \ V  V /  | (_) | | (_) | | | |  __/
 |______|  \__,_| |___/  \__, | |_____/    \_/\_/    \___/   \___/  |_|  \___|
                          __/ |
                         |___/

欢迎使用为API而生的 easySwoole 框架 当前版本: 3.x

使用:
  easyswoole [操作] [选项]

操作:
  install       安装easySwoole
  start         启动easySwoole
  stop          停止easySwoole
  reload        重启easySwoole
  help          查看命令的帮助信息

有关某个操作的详细信息 请使用 help 命令查看 
如查看 start 操作的详细信息 请输入 easyswoole help -start
```

## 服务启动
开发模式： 
```
php easyswoole start
```
## 守护模式启动
```
php easyswoole start d
```
## 生产环境(默认配置加载dev.env,使用该命令加载produce.env)
```
php easyswoole start produce
```
## 服务停止
```
php easyswoole stop
```
> 注意，守护模式下才需要stop，不然control+c或者是终端断开就退出进程了

## 服务热重启
```
php easyswoole reload 只重启task进程
php easyswoole reload all  重启task + worker进程
```
> 注意，守护模式下才需要reload，不然control+c或者是终端断开就退出进程了，此处为热重启，可以用于更新worker start后才加载的文件（业务逻辑），主进程（如配置文件）不会被重启。

# 热加载

热加载在开发阶段是非常有必要使用的，否则调试代码需要不停的重启服务，下面的脚本可以让我们只专注与开发，而不用去重启服务。

## mac OS

MacOS 下使用 `fswatch` 命令监听文件变更，然后重启服务器，需要先安装命令行工具 `brew install fswatch`

在程序根目录下创建文件`start.sh` 并 `chmod +x start.sh`

然后复制如下shell脚本保存到 `start.sh` 文件夹

```bash
#!/bin/bash
DIR=$1

if [ ! -n "$DIR" ] ;then
    echo "you have not choice Application directory !"
    exit
fi

php easyswoole stop
php easyswoole start d

fswatch -r $DIR | while read file
do
   echo "${file} was modify" >> ./Temp/reload.log 2>&1
   php easyswoole reload all
done
```
使用方法： `./start.sh ./App` 

如果直接执行 `./start.sh` 会提示 `you have not choice App directory`，因为我们需要指定监听路径，通常是`App` 目录


所以执行命令 `./start.sh ./App` 监听的路径为相对路径或绝对路径，相对路径注意使用 `./` 开头，否则会监听成 `Mac OS` 里 `/App` 目录。


启动后脚本会自动启动 `easyswoole` 并进入守护模式，但注意进程还是会hang住，因为 `fswatch` 会不断监听文件变更，如果 `Ctrl+c` 关闭进程则仅关闭了文件监听，`easyswoole` 会依然再后台运行。此时可以手动停止服务或者再次运行热加载脚本。
 
 
如果需要将热加载脚本也放入后台则使用命令 <code> nohup ./start.sh ./App &</code> 即可(注意最后有个and符号)。  

## Linux

**Linux和Mac Os 可以使用相同脚本，不过需要额外安装fswatch。**

*安装fswatch*  
> wget https://github.com/emcrisostomo/fswatch/releases/download/1.11.2/fswatch-1.11.2.tar.gz  
> tar -xvzf fswatch-1.11.2.tar.gz  
> cd fswatch-1.11.2  
> sudo ./configure  
> sudo make  
> sudo make install
> sudo ldconfig  

**确保动态库的安装目录($PREFIX/lib)包含在您的操作系统的动态链接器的查找路径中。默认路径/usr/local/lib.  
刷新链接和缓存到动态库是必需的。在GNU/Linux系统中，您可能需要运行 $ ldconfig**

脚本和上面的Mac OS的相同

**如果你运行脚本提示
> PID file does not exist, please check whether to run in the daemon mode!  
不必担心， 这个是脚本会先执行php easyswoole stop的缘故(因为你并没有启动easyswoole)**

## 文件扫描式热重启

当运行环境没有扩展或不想依赖脚本进行热重启时，可以新建如下的进程，通过循环扫描文件的最后修改时间，实现热重启

```php
<?php

namespace App\Process;

use EasySwoole\EasySwoole\ServerManager;
use EasySwoole\EasySwoole\Swoole\Process\AbstractProcess;
use EasySwoole\Utility\File;
use Swoole\Process;
use Swoole\Table;
use Swoole\Timer;

/**
 * 暴力热重载
 * Class HotReload
 * @package App\Process
 */
class HotReload extends AbstractProcess
{
    /** @var \swoole_table $table */
    protected $table;
    protected $isReady = false;

    /**
     * 启动定时器进行循环扫描
     * @param Process $process
     */
    public function run(Process $process)
    {
        $this->table = new Table(2048);
        $this->table->column('mtime', Table::TYPE_INT, 4);
        $this->table->create();
        $this->runComparison();
        Timer::tick(1000, function () {
            $this->runComparison();
        });
    }

    /**
     * 扫描文件变更
     */
    private function runComparison()
    {
        $startTime = microtime(true);
        $doReload = false;
        $files = File::scanDirectory(EASYSWOOLE_ROOT . '/App');
        if (isset($files['files'])) {
            foreach ($files['files'] as $file) {
                $currentTime = filemtime($file);
                $inode = crc32($file);
                if (!$this->table->exist($inode)) {
                    $doReload = true;
                    $this->table->set($inode, ['mtime' => $currentTime]);
                } else {
                    $oldTime = $this->table->get($inode)['mtime'];
                    if ($oldTime != $currentTime) {
                        $doReload = true;
                    }
                    $this->table->set($inode, ['mtime' => $currentTime]);
                }
            }
        }
        if ($doReload) {
            $count = $this->table->count();
            $time = date('Y-m-d H:i:s');
            $usage = round(microtime(true) - $startTime, 3);
            if (!$this->isReady == false) {
                echo "severReload at {$time} use : {$usage} s total: {$count} files\n";
                ServerManager::getInstance()->getSwooleServer()->reload();
            } else {
                echo "hot reload ready at {$time} use : {$usage} s total: {$count} files\n";
                $this->isReady = true;
            }
        }
    }

    public function onShutDown()
    {
        // TODO: Implement onShutDown() method.
    }

    public function onReceive(string $str)
    {
        // TODO: Implement onReceive() method.
    }
}
```

注意将命名空间对应自己的项目文件所在的路径，并在全局事件内注册该进程

```php
public static function mainServerCreate(EventRegister $register)
{
    ServerManager::getInstance()->getSwooleServer()->addProcess((new HotReload('HotReload'))->getProcess());
}
```
