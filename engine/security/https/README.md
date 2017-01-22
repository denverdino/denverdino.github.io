---
published: false
---

这是一个让测试[https.md](https.md)里例子更加容易的尝试。

现在还得手工做，不过我已经在boot2docker里跑起来了。

过程如下：

    $ boot2docker ssh
    root@boot2docker:/# git clone https://github.com/docker/docker
    root@boot2docker:/# cd docker/docs/articles/https
    root@boot2docker:/# make cert


有很多输出和要手工输入的问题，openssl需要交互式运行。

**注意:** 注意当提示`Computer Name`时你输入的是hostname(我的例子里是`boot2docker`)

    root@boot2docker:/# sudo make run

启动另外一个终端:

    $ boot2docker ssh
    root@boot2docker:/# cd docker/docs/articles/https
    root@boot2docker:/# make client

最后，先用`--tls`连接，再用`--tlsverify`连接，应该都成功。
