[English](./README.md) | 中文

# Swoole

[![Latest Version](https://img.shields.io/github/release/swoole/swoole-src.svg?style=flat-square)](https://github.com/swoole/swoole-src/releases)
[![Build Status](https://api.travis-ci.org/swoole/swoole-src.svg)](https://travis-ci.org/swoole/swoole-src)
[![License](https://img.shields.io/badge/license-apache2-blue.svg)](LICENSE)
[![Join the chat at https://gitter.im/swoole/swoole-src](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/swoole/swoole-src?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Coverity Scan Build Status](https://scan.coverity.com/projects/11654/badge.svg)](https://scan.coverity.com/projects/swoole-swoole-src)
[![Backers on Open Collective](https://opencollective.com/swoole-src/backers/badge.svg)](#backers) 
[![Sponsors on Open Collective](https://opencollective.com/swoole-src/sponsors/badge.svg)](#sponsors) 

![](./mascot.png)

**Swoole是一个为PHP用C和C++编写的基于事件的高性能异步&协程并行网络通信引擎**

## ✨事件驱动

Swoole中的网络请求处理是基于事件的，并且充分利用了底层的epoll / kqueue实现，使得为数百万个请求提供服务变得非常容易。

Swoole4使用全新的协程内核引擎，现在它拥有一个全职的开发团队，因此我们正在进入PHP历史上前所未有的时期，为性能的高速提升提供了独一无二的可能性。

## ⚡️协程

Swoole4或更高版本拥有高可用性的内置协程，您可以使用完全同步的代码来实现异步性能，PHP代码没有任何额外的关键字，底层会自动进行协程调度。

开发者可以将协程理解为超轻量级的线程, 你可以非常容易地在一个进程中创建成千上万个协程。

### MySQL客户端

并发1万个请求从MySQL读取海量数据仅需要0.2秒

```php
$s = microtime(true);
for ($c = 100; $c--;) {
    go(function () {
        $mysql = new Swoole\Coroutine\MySQL;
        $mysql->connect([
            'host' => '127.0.0.1',
            'user' => 'root',
            'password' => 'root',
            'database' => 'test'
        ]);
        $statement = $mysql->prepare('SELECT * FROM `user`');
        for ($n = 100; $n--;) {
            $result = $statement->execute();
            assert(count($result) > 0);
        }
    });
}
Swoole\Event::wait();
echo 'use ' . (microtime(true) - $s) . ' s';
```

### 混合服务器

你可以在一个事件循环上创建多个服务：TCP，HTTP，Websocket和HTTP2，并且能轻松承载上万请求。

```php
function tcp_pack(string $data): string
{
    return pack('N', strlen($data)) . $data;
}
function tcp_unpack(string $data): string
{
    return substr($data, 4, unpack('N', substr($data, 0, 4))[1]);
}
$tcp_options = [
    'open_length_check' => true,
    'package_length_type' => 'N',
    'package_length_offset' => 0,
    'package_body_offset' => 4
];
```

```php
$server = new Swoole\WebSocket\Server('127.0.0.1', 9501, SWOOLE_BASE);
$server->set(['open_http2_protocol' => true]);
// http && http2
$server->on('request', function (Swoole\Http\Request $request, Swoole\Http\Response $response) {
    $response->end('Hello ' . $request->rawcontent());
});
// websocket
$server->on('message', function (Swoole\WebSocket\Server $server, Swoole\WebSocket\Frame $frame) {
    $server->push($frame->fd, 'Hello ' . $frame->data);
});
// tcp
$tcp_server = $server->listen('127.0.0.1', 9502, SWOOLE_TCP);
$tcp_server->set($tcp_options);
$tcp_server->on('receive', function (Swoole\Server $server, int $fd, int $reactor_id, string $data) {
    $server->send($fd, tcp_pack('Hello ' . tcp_unpack($data)));
});
$server->start();
```
### 多种客户端

不管是DNS查询抑或是发送请求和接收响应，都是协程调度的，不会产生任何阻塞。

```php
go(function () {
    // http
    $http_client = new Swoole\Coroutine\Http\Client('127.0.0.1', 9501);
    assert($http_client->post('/', 'Swoole Http'));
    var_dump($http_client->body);
    // websocket
    $http_client->upgrade('/');
    $http_client->push('Swoole Websocket');
    var_dump($http_client->recv()->data);
});
go(function () {
    // http2
    $http2_client = new Swoole\Coroutine\Http2\Client('localhost', 9501);
    $http2_client->connect();
    $http2_request = new Swoole\Http2\Request;
    $http2_request->method = 'POST';
    $http2_request->data = 'Swoole Http2';
    $http2_client->send($http2_request);
    $http2_response = $http2_client->recv();
    var_dump($http2_response->data);
});
go(function () use ($tcp_options) {
    // tcp
    $tcp_client = new Swoole\Coroutine\Client(SWOOLE_TCP);
    $tcp_client->set($tcp_options);
    $tcp_client->connect('127.0.0.1', 9502);
    $tcp_client->send(tcp_pack('Swoole Tcp'));
    var_dump(tcp_unpack($tcp_client->recv()));
});
```

### 通道

通道(Channel)是协程之间通信交换数据的唯一渠道, 而协程+通道的开发组合即为著名的CSP编程模型。

在Swoole开发中，Channel常用于连接池的实现和协程并发的调度。

#### 连接池最简示例

在以下示例中，我们并发了一千个redis请求，通常的情况下，这已经超过了Redis最大的连接数，将会抛出连接异常， 但基于Channel实现的连接池可以完美地调度请求，开发者就无需担心连接过载。

```php
class RedisPool
{
    /**@var \Swoole\Coroutine\Channel */
    protected $pool;

    /**
     * RedisPool constructor.
     * @param int $size max connections
     */
    public function __construct(int $size = 100)
    {
        $this->pool = new \Swoole\Coroutine\Channel($size);
        for ($i = 0; $i < $size; $i++) {
            $redis = new \Swoole\Coroutine\Redis();
            $res = $redis->connect('127.0.0.1', 6379);
            if ($res == false) {
                throw new \RuntimeException("failed to connect redis server.");
            } else {
                $this->put($redis);
            }
        }
    }

    public function get(): \Swoole\Coroutine\Redis
    {
        return $this->pool->pop();
    }

    public function put(\Swoole\Coroutine\Redis $redis)
    {
        $this->pool->push($redis);
    }

    public function close(): void
    {
        $this->pool->close();
        $this->pool = null;
    }
}

go(function () {
    $pool = new RedisPool();
    // max concurrency num is more than max connections
    // but it's no problem, channel will help you with scheduling
    for ($c = 0; $c < 1000; $c++) {
        go(function () use ($pool, $c) {
            for ($n = 0; $n < 100; $n++) {
                $redis = $pool->get();
                assert($redis->set("awesome-{$c}-{$n}", 'swoole'));
                assert($redis->get("awesome-{$c}-{$n}") === 'swoole');
                assert($redis->delete("awesome-{$c}-{$n}"));
                $pool->put($redis);
            }
        });
    }
});
```

#### 生产和消费

Swoole的部分客户端实现了defer机制来进行并发，但你依然可以用协程和通道的组合来灵活地实现它。

```php
go(function () {
    // User: I need you to bring me some information back.
    // Channel: OK! I will be responsible for scheduling.
    $channel = new Swoole\Coroutine\Channel;
    go(function () use ($channel) {
        // Coroutine A: Ok! I will show you the github addr info
        $addr_info = Co::getaddrinfo('github.com');
        $channel->push(['A', json_encode($addr_info, JSON_PRETTY_PRINT)]);
    });
    go(function () use ($channel) {
        // Coroutine B: Ok! I will show you what your code look like
        $mirror = Co::readFile(__FILE__);
        $channel->push(['B', $mirror]);
    });
    go(function () use ($channel) {
        // Coroutine C: Ok! I will show you the date
        $channel->push(['C', date(DATE_W3C)]);
    });
    for ($i = 3; $i--;) {
        list($id, $data) = $channel->pop();
        echo "From {$id}:\n {$data}\n";
    }
    // User: Amazing, I got every information at earliest time!
});
```

### 定时器

```php
$id = Swoole\Timer::tick(100, function () {
    echo "⚙️ Do something...\n";
});
Swoole\Timer::after(500, function () use ($id) {
    Swoole\Timer::clear($id);
    echo "⏰ Done\n";
});
Swoole\Timer::after(1000, function () use ($id) {
    if (!Swoole\Timer::exists($id)) {
        echo "✅ All right!\n";
    }
});
```

#### 使用协程方式
```php
go(function () {
    $i = 0;
    while (true) {
        Co::sleep(0.1);
        echo "📝 Do something...\n";
        if (++$i === 5) {
            echo "🛎 Done\n";
            break;
        }
    }
    echo "🎉 All right!\n";
});
```

### 命名空间

Swoole提供了多种类命名规则以满足不同开发者的爱好

1. 符合PSR规范的命名空间风格
2. 便于键入的下划线风格
3. 协程类短名风格

## 🔥 强大的运行时钩子

在最新版本的Swoole中，我们添加了一项新功能，使PHP原生的同步网络库一键化成为协程库。

只需在脚本顶部调用`Swoole\Runtime::enableCoroutine()`方法并使用`php-redis`，并发1万个请求从Redis读取数据仅需0.1秒！

```php
Swoole\Runtime::enableCoroutine();
$s = microtime(true);
for ($c = 100; $c--;) {
    go(function () {
        ($redis = new Redis)->connect('127.0.0.1', 6379);
        for ($n = 100; $n--;) {
            assert($redis->get('awesome') === 'swoole');
        }
    });
}
Swoole\Event::wait();
echo 'use ' . (microtime(true) - $s) . ' s';
```

调用它之后，Swoole内核将替换ZendVM中的Stream函数指针，如果使用基于`php_stream`的扩展，则所有套接字操作都可以在运行时动态转换为协程调度的异步IO。

### 你可以在一秒钟里做多少事?

睡眠1万次，读取，写入，检查和删除文件1万次，使用PDO和MySQLi与数据库通信1万次，创建TCP服务器和多个客户端相互通信1万次，创建UDP服务器和多个客户端到相互通信1万次......一切都在一个进程中完美完成！

```php
Swoole\Runtime::enableCoroutine();
$s = microtime(true);

// i just want to sleep...
for ($c = 100; $c--;) {
    go(function () {
        for ($n = 100; $n--;) {
            usleep(1000);
        }
    });
}

// 10k file read and write
for ($c = 100; $c--;) {
    go(function () use ($c) {
        $tmp_filename = "/tmp/test-{$c}.php";
        for ($n = 100; $n--;) {
            $self = file_get_contents(__FILE__);
            file_put_contents($tmp_filename, $self);
            assert(file_get_contents($tmp_filename) === $self);
        }
        unlink($tmp_filename);
    });
}

// 10k pdo and mysqli read
for ($c = 50; $c--;) {
    go(function () {
        $pdo = new PDO('mysql:host=127.0.0.1;dbname=test;charset=utf8', 'root', 'root');
        $statement = $pdo->prepare('SELECT * FROM `user`');
        for ($n = 100; $n--;) {
            $statement->execute();
            assert(count($statement->fetchAll()) > 0);
        }
    });
}
for ($c = 50; $c--;) {
    go(function () {
        $mysqli = new Mysqli('127.0.0.1', 'root', 'root', 'test');
        $statement = $mysqli->prepare('SELECT `id` FROM `user`');
        for ($n = 100; $n--;) {
            $statement->bind_result($id);
            $statement->execute();
            $statement->fetch();
            assert($id > 0);
        }
    });
}

// php_stream tcp server & client with 12.8k requests in single process
function tcp_pack(string $data): string
{
    return pack('n', strlen($data)) . $data;
}

function tcp_length(string $head): int
{
    return unpack('n', $head)[1];
}

go(function () {
    $ctx = stream_context_create(['socket' => ['so_reuseaddr' => true, 'backlog' => 128]]);
    $socket = stream_socket_server(
        'tcp://0.0.0.0:9502',
        $errno, $errstr, STREAM_SERVER_BIND | STREAM_SERVER_LISTEN, $ctx
    );
    if (!$socket) {
        echo "$errstr ($errno)\n";
    } else {
        $i = 0;
        while ($conn = stream_socket_accept($socket, 1)) {
            stream_set_timeout($conn, 5);
            for ($n = 100; $n--;) {
                $data = fread($conn, tcp_length(fread($conn, 2)));
                assert($data === "Hello Swoole Server #{$n}!");
                fwrite($conn, tcp_pack("Hello Swoole Client #{$n}!"));
            }
            if (++$i === 128) {
                fclose($socket);
                break;
            }
        }
    }
});
for ($c = 128; $c--;) {
    go(function () {
        $fp = stream_socket_client("tcp://127.0.0.1:9502", $errno, $errstr, 1);
        if (!$fp) {
            echo "$errstr ($errno)\n";
        } else {
            stream_set_timeout($fp, 5);
            for ($n = 100; $n--;) {
                fwrite($fp, tcp_pack("Hello Swoole Server #{$n}!"));
                $data = fread($fp, tcp_length(fread($fp, 2)));
                assert($data === "Hello Swoole Client #{$n}!");
            }
            fclose($fp);
        }
    });
}

// udp server & client with 12.8k requests in single process
go(function () {
    $socket = new Swoole\Coroutine\Socket(AF_INET, SOCK_DGRAM, 0);
    $socket->bind('127.0.0.1', 9503);
    $client_map = [];
    for ($c = 128; $c--;) {
        for ($n = 0; $n < 100; $n++) {
            $recv = $socket->recvfrom($peer);
            $client_uid = "{$peer['address']}:{$peer['port']}";
            $id = $client_map[$client_uid] = ($client_map[$client_uid] ?? -1) + 1;
            assert($recv === "Client: Hello #{$id}!");
            $socket->sendto($peer['address'], $peer['port'], "Server: Hello #{$id}!");
        }
    }
    $socket->close();
});
for ($c = 128; $c--;) {
    go(function () {
        $fp = stream_socket_client("udp://127.0.0.1:9503", $errno, $errstr, 1);
        if (!$fp) {
            echo "$errstr ($errno)\n";
        } else {
            for ($n = 0; $n < 100; $n++) {
                fwrite($fp, "Client: Hello #{$n}!");
                $recv = fread($fp, 1024);
                list($address, $port) = explode(':', (stream_socket_get_name($fp, true)));
                assert($address === '127.0.0.1' && (int)$port === 9503);
                assert($recv === "Server: Hello #{$n}!");
            }
            fclose($fp);
        }
    });
}

Swoole\Event::wait();
echo 'use ' . (microtime(true) - $s) . ' s';
```

## ⌛️ 安装

> 和任何开源项目一样, Swoole总是在**最新的发行版**提供最可靠的稳定性和最强的功能, 请尽量保证你使用的是最新版本

### 1. 直接使用Swoole官方的二进制包 (初学者 + 开发环境)

访问我们官网的[下载页面](https://www.swoole.com/page/download)

### 编译需求

- Linux, OS X 系统 或 CygWin, WSL
- PHP 7.0.0 或以上版本 (版本越高性能越好)
- GCC 4.8 及以上

### 2. 使用PHP官方的PECL工具安装 (初学者)

```shell
pecl install swoole
```

### 3. 从源码编译安装 (推荐)

> 非内核开发研究之用途, 请下载[发布版本](https://github.com/swoole/swoole-src/releases)的源码编译

```shell
cd swoole-src && \
phpize && \
./configure && \
make && sudo make install
```

#### 启用扩展

编译安装到系统成功后, 需要在`php.ini`中加入一行`extension=swoole.so`来启用Swoole扩展

#### 额外编译参数

> 使用例子: `./configure --enable-openssl --enable-sockets`  (`sudo yum/apt install openssl  && cp xxxxxx/openssl ./include/  -r`)

- `--enable-openssl` 或 `--with-openssl-dir=./openssl-1.0.2k`     (`wget http://ftp.openssl.org/source/old/1.0.2/openssl-1.0.2k.tar.gz .  && tar openssl-1.0.2k.tar.gz && `)
- `--enable-sockets`
- `--enable-http2`
- `--enable-mysqlnd` (需要 mysqlnd, 只是为了支持`mysql->escape`方法)

### 升级

>  ⚠️ 如果你要从源码升级, 别忘记在源码目录执行 `make clean` 

1. `pecl upgrade swoole`
2. `git pull && cd swoole-src && make clean && make && sudo make install`
3. 如果你改变了PHP版本, 请重新执行 `phpize clean && phpize`后重新编译

## 💎 框架 & 组件

- [**Hyperf**](https://github.com/hyperf-cloud/hyperf) 是一个高性能、高灵活性的协程框架，存在丰富的可能性，如实现分布式中间件，微服务架构等
- [**Swoft**](https://github.com/swoft-cloud) 是一个现代化的面向切面的高性能协程全栈组件化框架
- [**Easyswoole**](https://www.easyswoole.com) 是一个极简的高性能的框架，让代码开发就好像写`echo "hello world"`一样简单
- [**Saber**](https://github.com/swlib/saber) 是一个人性化的高性能HTTP客户端组件，几乎拥有一切你可以想象的强大功能

## 🛠 开发 & 讨论

- __中文文档__: <http://wiki.swoole.com>
- __Document__: <https://www.swoole.co.uk/docs>
- __IDE Helper & API__: <https://github.com/swoole/ide-helper>
- __中文社区及QQ群__: <https://wiki.swoole.com/wiki/page/p-discussion.html>
- __Twitter__: <https://twitter.com/php_swoole>
- __Slack Group__: <https://swoole.slack.com>

## 🍭 性能测试

+ 在开源的 [Techempower Web Framework benchmarks](https://www.techempower.com/benchmarks/#section=data-r17) 压测平台上，Swoole使用MySQL数据库压测的成绩一度位居首位， 所有IO性能测试都位列第一梯队。
+ 你可以直接运行[Benchmark Script](./benchmark/benchmark.php)来快速地测试出Swoole提供的Http服务在你的机器上所能达到的最大QPS

## 🖊️ 如何贡献

非常欢迎您对Swoole的开发作出贡献！

你可以选择以下方式向Swoole贡献：

- [发布issue进行问题反馈和建议](https://github.com/swoole/swoole-src/issues)
- 通过Pull Request提交修复
- 完善我们的文档和例子

## ❤️ 贡献者

项目的发展离不开以下贡献者的努力! [[Contributor](https://github.com/swoole/swoole-src/graphs/contributors)].
<a href="https://github.com/swoole/swoole-src/graphs/contributors"><img src="https://opencollective.com/swoole-src/contributors.svg?width=890&button=false" /></a>

## 📃 开源协议

Apache License Version 2.0 see http://www.apache.org/licenses/LICENSE-2.0.html




English | [中文](./README-CN.md)

Swoole
======
[![Latest Version](https://img.shields.io/github/release/swoole/swoole-src.svg?style=flat-square)](https://github.com/swoole/swoole-src/releases)
[![Build Status](https://api.travis-ci.org/swoole/swoole-src.svg)](https://travis-ci.org/swoole/swoole-src)
[![License](https://img.shields.io/badge/license-apache2-blue.svg)](LICENSE)
[![Join the chat at https://gitter.im/swoole/swoole-src](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/swoole/swoole-src?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Coverity Scan Build Status](https://scan.coverity.com/projects/11654/badge.svg)](https://scan.coverity.com/projects/swoole-swoole-src)
[![Backers on Open Collective](https://opencollective.com/swoole-src/backers/badge.svg)](#backers) 
[![Sponsors on Open Collective](https://opencollective.com/swoole-src/sponsors/badge.svg)](#sponsors) 

![](./mascot.png)

**Swoole is an event-driven asynchronous & coroutine-based concurrency networking communication engine with high performance written in C and C++ for PHP.**

## ✨Event-based

The network layer in Swoole is event-based and takes full advantage of the underlying epoll/kqueue implementation, making it really easy to serve millions of requests.

Swoole 4.x use a brand new engine kernel and now it has a full-time developer team, so we are entering an unprecedented period in PHP history which offers a unique possibility for rapid evolution in performance.

## ⚡️Coroutine

Swoole 4.x or later supports the built-in coroutine with high availability, and you can use fully synchronized code to implement asynchronous performance. PHP code without any additional keywords, the underlying automatic coroutine-scheduling.

Developers can understand coroutines as ultra-lightweight threads, and you can easily create thousands of coroutines in a single process.

### MySQL

Concurrency 10K requests to read data from MySQL takes only 0.2s!

```php
$s = microtime(true);
for ($c = 100; $c--;) {
    go(function () {
        $mysql = new Swoole\Coroutine\MySQL;
        $mysql->connect([
            'host' => '127.0.0.1',
            'user' => 'root',
            'password' => 'root',
            'database' => 'test'
        ]);
        $statement = $mysql->prepare('SELECT * FROM `user`');
        for ($n = 100; $n--;) {
            $result = $statement->execute();
            assert(count($result) > 0);
        }
    });
}
Swoole\Event::wait();
echo 'use ' . (microtime(true) - $s) . ' s';
```

### Mixed server

You can create multiple services on the single event loop: TCP, HTTP, Websocket and HTTP2, and easily handle thousands of requests.

```php
function tcp_pack(string $data): string
{
    return pack('N', strlen($data)) . $data;
}
function tcp_unpack(string $data): string
{
    return substr($data, 4, unpack('N', substr($data, 0, 4))[1]);
}
$tcp_options = [
    'open_length_check' => true,
    'package_length_type' => 'N',
    'package_length_offset' => 0,
    'package_body_offset' => 4
];
```

```php
$server = new Swoole\WebSocket\Server('127.0.0.1', 9501, SWOOLE_BASE);
$server->set(['open_http2_protocol' => true]);
// http && http2
$server->on('request', function (Swoole\Http\Request $request, Swoole\Http\Response $response) {
    $response->end('Hello ' . $request->rawcontent());
});
// websocket
$server->on('message', function (Swoole\WebSocket\Server $server, Swoole\WebSocket\Frame $frame) {
    $server->push($frame->fd, 'Hello ' . $frame->data);
});
// tcp
$tcp_server = $server->listen('127.0.0.1', 9502, SWOOLE_TCP);
$tcp_server->set($tcp_options);
$tcp_server->on('receive', function (Swoole\Server $server, int $fd, int $reactor_id, string $data) {
    $server->send($fd, tcp_pack('Hello ' . tcp_unpack($data)));
});
$server->start();
```

### Coroutine clients

Whether you DNS query or send requests or receive responses, all of these are scheduled by coroutine automatically.

```php
go(function () {
    // http
    $http_client = new Swoole\Coroutine\Http\Client('127.0.0.1', 9501);
    assert($http_client->post('/', 'Swoole Http'));
    var_dump($http_client->body);
    // websocket
    $http_client->upgrade('/');
    $http_client->push('Swoole Websocket');
    var_dump($http_client->recv()->data);
});
go(function () {
    // http2
    $http2_client = new Swoole\Coroutine\Http2\Client('localhost', 9501);
    $http2_client->connect();
    $http2_request = new Swoole\Http2\Request;
    $http2_request->method = 'POST';
    $http2_request->data = 'Swoole Http2';
    $http2_client->send($http2_request);
    $http2_response = $http2_client->recv();
    var_dump($http2_response->data);
});
go(function () use ($tcp_options) {
    // tcp
    $tcp_client = new Swoole\Coroutine\Client(SWOOLE_TCP);
    $tcp_client->set($tcp_options);
    $tcp_client->connect('127.0.0.1', 9502);
    $tcp_client->send(tcp_pack('Swoole Tcp'));
    var_dump(tcp_unpack($tcp_client->recv()));
});
```

### Channel

Channel is the only way for exchanging data between coroutines, the development combination of the `Coroutine + Channel` is the famous CSP programming model.

In Swoole development, Channel is usually used for implementing connection pool or scheduling coroutine concurrent.

#### The simplest example of a connection pool

In the following example, we have a thousand concurrently requests to redis. Normally, this has exceeded the maximum number of Redis connections setting and will throw a connection exception, but the connection pool based on Channel can perfectly schedule requests. We don't have to worry about connection overload.

```php
class RedisPool
{
    /**@var \Swoole\Coroutine\Channel */
    protected $pool;

    /**
     * RedisPool constructor.
     * @param int $size max connections
     */
    public function __construct(int $size = 100)
    {
        $this->pool = new \Swoole\Coroutine\Channel($size);
        for ($i = 0; $i < $size; $i++) {
            $redis = new \Swoole\Coroutine\Redis();
            $res = $redis->connect('127.0.0.1', 6379);
            if ($res == false) {
                throw new \RuntimeException("failed to connect redis server.");
            } else {
                $this->put($redis);
            }
        }
    }

    public function get(): \Swoole\Coroutine\Redis
    {
        return $this->pool->pop();
    }

    public function put(\Swoole\Coroutine\Redis $redis)
    {
        $this->pool->push($redis);
    }

    public function close(): void
    {
        $this->pool->close();
        $this->pool = null;
    }
}

go(function () {
    $pool = new RedisPool();
    // max concurrency num is more than max connections
    // but it's no problem, channel will help you with scheduling
    for ($c = 0; $c < 1000; $c++) {
        go(function () use ($pool, $c) {
            for ($n = 0; $n < 100; $n++) {
                $redis = $pool->get();
                assert($redis->set("awesome-{$c}-{$n}", 'swoole'));
                assert($redis->get("awesome-{$c}-{$n}") === 'swoole');
                assert($redis->delete("awesome-{$c}-{$n}"));
                $pool->put($redis);
            }
        });
    }
});
```

#### Producer and consumers

Some Swoole's clients implement the defer mode for concurrency, but you can still implement it flexible with a combination of coroutines and channels.

```php
go(function () {
    // User: I need you to bring me some information back.
    // Channel: OK! I will be responsible for scheduling.
    $channel = new Swoole\Coroutine\Channel;
    go(function () use ($channel) {
        // Coroutine A: Ok! I will show you the github addr info
        $addr_info = Co::getaddrinfo('github.com');
        $channel->push(['A', json_encode($addr_info, JSON_PRETTY_PRINT)]);
    });
    go(function () use ($channel) {
        // Coroutine B: Ok! I will show you what your code look like
        $mirror = Co::readFile(__FILE__);
        $channel->push(['B', $mirror]);
    });
    go(function () use ($channel) {
        // Coroutine C: Ok! I will show you the date
        $channel->push(['C', date(DATE_W3C)]);
    });
    for ($i = 3; $i--;) {
        list($id, $data) = $channel->pop();
        echo "From {$id}:\n {$data}\n";
    }
    // User: Amazing, I got every information at earliest time!
});
```

### Timer

```php
$id = Swoole\Timer::tick(100, function () {
    echo "⚙️ Do something...\n";
});
Swoole\Timer::after(500, function () use ($id) {
    Swoole\Timer::clear($id);
    echo "⏰ Done\n";
});
Swoole\Timer::after(1000, function () use ($id) {
    if (!Swoole\Timer::exists($id)) {
        echo "✅ All right!\n";
    }
});
```
#### The way of coroutine

```php
go(function () {
    $i = 0;
    while (true) {
        Co::sleep(0.1);
        echo "📝 Do something...\n";
        if (++$i === 5) {
            echo "🛎 Done\n";
            break;
        }
    }
    echo "🎉 All right!\n";
});
```

## 🔥 Amazing runtime hooks

**As of Swoole v4.1.0, we added the ability to transform synchronous PHP network libraries into co-routine libraries using a single line of code.**

Simply call the `Swoole\Runtime::enableCoroutine()` method at the top of your script. In the sample below we connect to php-redis and concurrently read 10k requests in 0.1s:

```php
Swoole\Runtime::enableCoroutine();
$s = microtime(true);
for ($c = 100; $c--;) {
    go(function () {
        ($redis = new Redis)->connect('127.0.0.1', 6379);
        for ($n = 100; $n--;) {
            assert($redis->get('awesome') === 'swoole');
        }
    });
}
Swoole\Event::wait();
echo 'use ' . (microtime(true) - $s) . ' s';
```

By calling this method, the Swoole kernel replaces ZendVM stream function pointers. If you use `php_stream` based extensions, all socket operations can be dynamically converted to be asynchronous IO scheduled by coroutine at runtime!

### How many things you can do in 1s?

Sleep 10K times, read, write, check and delete files 10K times, use PDO and MySQLi to communicate with the database 10K times, create a TCP server and multiple clients to communicate with each other 10K times, create a UDP server and multiple clients to communicate with each other 10K times... Everything works well in one process!

Just see what the Swoole brings, just imagine...

```php
Swoole\Runtime::enableCoroutine();
$s = microtime(true);

// i just want to sleep...
for ($c = 100; $c--;) {
    go(function () {
        for ($n = 100; $n--;) {
            usleep(1000);
        }
    });
}

// 10K file read and write
for ($c = 100; $c--;) {
    go(function () use ($c) {
        $tmp_filename = "/tmp/test-{$c}.php";
        for ($n = 100; $n--;) {
            $self = file_get_contents(__FILE__);
            file_put_contents($tmp_filename, $self);
            assert(file_get_contents($tmp_filename) === $self);
        }
        unlink($tmp_filename);
    });
}

// 10K pdo and mysqli read
for ($c = 50; $c--;) {
    go(function () {
        $pdo = new PDO('mysql:host=127.0.0.1;dbname=test;charset=utf8', 'root', 'root');
        $statement = $pdo->prepare('SELECT * FROM `user`');
        for ($n = 100; $n--;) {
            $statement->execute();
            assert(count($statement->fetchAll()) > 0);
        }
    });
}
for ($c = 50; $c--;) {
    go(function () {
        $mysqli = new Mysqli('127.0.0.1', 'root', 'root', 'test');
        $statement = $mysqli->prepare('SELECT `id` FROM `user`');
        for ($n = 100; $n--;) {
            $statement->bind_result($id);
            $statement->execute();
            $statement->fetch();
            assert($id > 0);
        }
    });
}

// php_stream tcp server & client with 12.8K requests in single process
function tcp_pack(string $data): string
{
    return pack('n', strlen($data)) . $data;
}

function tcp_length(string $head): int
{
    return unpack('n', $head)[1];
}

go(function () {
    $ctx = stream_context_create(['socket' => ['so_reuseaddr' => true, 'backlog' => 128]]);
    $socket = stream_socket_server(
        'tcp://0.0.0.0:9502',
        $errno, $errstr, STREAM_SERVER_BIND | STREAM_SERVER_LISTEN, $ctx
    );
    if (!$socket) {
        echo "$errstr ($errno)\n";
    } else {
        $i = 0;
        while ($conn = stream_socket_accept($socket, 1)) {
            stream_set_timeout($conn, 5);
            for ($n = 100; $n--;) {
                $data = fread($conn, tcp_length(fread($conn, 2)));
                assert($data === "Hello Swoole Server #{$n}!");
                fwrite($conn, tcp_pack("Hello Swoole Client #{$n}!"));
            }
            if (++$i === 128) {
                fclose($socket);
                break;
            }
        }
    }
});
for ($c = 128; $c--;) {
    go(function () {
        $fp = stream_socket_client("tcp://127.0.0.1:9502", $errno, $errstr, 1);
        if (!$fp) {
            echo "$errstr ($errno)\n";
        } else {
            stream_set_timeout($fp, 5);
            for ($n = 100; $n--;) {
                fwrite($fp, tcp_pack("Hello Swoole Server #{$n}!"));
                $data = fread($fp, tcp_length(fread($fp, 2)));
                assert($data === "Hello Swoole Client #{$n}!");
            }
            fclose($fp);
        }
    });
}

// udp server & client with 12.8K requests in single process
go(function () {
    $socket = new Swoole\Coroutine\Socket(AF_INET, SOCK_DGRAM, 0);
    $socket->bind('127.0.0.1', 9503);
    $client_map = [];
    for ($c = 128; $c--;) {
        for ($n = 0; $n < 100; $n++) {
            $recv = $socket->recvfrom($peer);
            $client_uid = "{$peer['address']}:{$peer['port']}";
            $id = $client_map[$client_uid] = ($client_map[$client_uid] ?? -1) + 1;
            assert($recv === "Client: Hello #{$id}!");
            $socket->sendto($peer['address'], $peer['port'], "Server: Hello #{$id}!");
        }
    }
    $socket->close();
});
for ($c = 128; $c--;) {
    go(function () {
        $fp = stream_socket_client("udp://127.0.0.1:9503", $errno, $errstr, 1);
        if (!$fp) {
            echo "$errstr ($errno)\n";
        } else {
            for ($n = 0; $n < 100; $n++) {
                fwrite($fp, "Client: Hello #{$n}!");
                $recv = fread($fp, 1024);
                list($address, $port) = explode(':', (stream_socket_get_name($fp, true)));
                assert($address === '127.0.0.1' && (int)$port === 9503);
                assert($recv === "Server: Hello #{$n}!");
            }
            fclose($fp);
        }
    });
}

Swoole\Event::wait();
echo 'use ' . (microtime(true) - $s) . ' s';
```

## ⌛️ Installation

> As with any open source project, Swoole always provides the most reliable stability and the most powerful features in **the latest released version**. Please ensure as much as possible that you are using the latest version.

### 1. From binary package (beginners + dev-env)

See our [download page](https://www.swoole.com/page/download)

### Compiling requirements

- Linux, OS X or Cygwin, WSL
- PHP 7.0.0 or later (The higher the version, the better the performance.)
- GCC 4.8 or later

### 2. Install via PECL (beginners)

```shell
pecl install swoole
```

### 3. Install from source (recommended)

Please download the source packages from [Releases](https://github.com/swoole/swoole-src/releases) or:

```shell
git clone https://github.com/swoole/swoole-src.git && \
cd swoole-src && \
git checkout v4.x.x
```

Compile and install at the source folder:

```shell
phpize && \
./configure && \
make && make install
```

#### Enable extension in PHP

After compiling and installing to the system successfully, you have to add a new line `extension=swoole.so` to `php.ini` to enable Swoole extension.

#### Extra compiler configurations

> for example: `./configure --enable-openssl --enable-sockets`

- `--enable-openssl` or `--with-openssl-dir=DIR`
- `--enable-sockets`
- `--enable-http2`
- `--enable-mysqlnd` (need mysqlnd, it just for supporting `$mysql->escape` method)

### Upgrade

>  ⚠️ If you upgrade from source, don't forget to `make clean` before you upgrade your swoole

1. `pecl upgrade swoole`
2. `git pull && cd swoole-src && make clean && make && sudo make install`
3. if you change your PHP version, please re-run `phpize clean && phpize` then try to compile


### Major change since version 4.3.0

Async clients and API are moved to a separate PHP extension `swoole_async` since version 4.3.0, install `swoole_async`:

```shell
git clone https://github.com/swoole/async-ext.git
cd async-src
phpize
./configure
make -j 4
sudo make install
```

Enable it by adding a new line `extension=swoole_async.so` to `php.ini`.

## 💎 Frameworks & Components
- [**Hyperf**](https://github.com/hyperf-cloud/hyperf) is a coroutine framework that focuses on hyperspeed and flexibility, specifically used for build microservices or middlewares.
- [**Swoft**](https://github.com/swoft-cloud/swoft) is a modern, high-performance AOP and coroutine PHP framework.
- [**Easyswoole**](https://www.easyswoole.com) is a simple, high-performance PHP framework, based on Swoole, which makes using Swoole as easy as `echo "hello world"`.
- [**Saber**](https://github.com/swlib/saber) Is a human-friendly, high-performance HTTP client component that has almost everything you can imagine.

## 🛠 Develop & Discussion

* __中文文档__: <http://wiki.swoole.com>
* __Documentation__: <https://www.swoole.co.uk/docs>
* __IDE Helper & API__: <https://github.com/swoole/ide-helper>
* __中文社区__: <https://wiki.swoole.com/wiki/page/p-discussion.html>
* __Twitter__: <https://twitter.com/php_swoole>
* __Slack Group__: <https://swoole.slack.com>

## 🍭 Benchmark

+ On the open source [Techempower Web Framework benchmarks](https://www.techempower.com/benchmarks/#section=data-r17) Swoole used MySQL database benchmark to rank first, and all performance tests ranked in the first echelon.
+ You can just run [Benchmark Script](./benchmark/benchmark.php) to quickly test the maximum QPS of Swoole-HTTP-Server on your machine.

## 🖊️ Security issues

Security issues should be reported privately, via email, to the Swoole develop team team@swoole.com. You should receive a response within 24 hours. If for some reason you do not, please follow up via email to ensure we received your original message.

## 🖊️ Contribution

Your contribution to Swoole development is very welcome!

You may contribute in the following ways:

* [Report issues and feedback](https://github.com/swoole/swoole-src/issues)
* Submit fixes, features via Pull Request
* Write/polish documentation

## ❤️ Contributors

This project exists thanks to all the people who contribute. [[Contributors](https://github.com/swoole/swoole-src/graphs/contributors)].
<a href="https://github.com/swoole/swoole-src/graphs/contributors"><img src="https://opencollective.com/swoole-src/contributors.svg?width=890&button=false" /></a>

## 🎙️ Official Evangelist

[Demin](https://deminy.in) has been playing with PHP since 2000, focusing on building high-performance, secure web services. He is an occasional conference speaker on PHP and Swoole, and has been working for companies in the states like eBay, Visa and Glu Mobile for years. You may find Demin on [Twitter](https://twitter.com/deminy) or [GitHub](https://github.com/deminy).

## 📃 License

Apache License Version 2.0 see http://www.apache.org/licenses/LICENSE-2.0.html
