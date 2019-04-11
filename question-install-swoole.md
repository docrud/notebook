# mac os 上安装 swoole 遇到的问题整理

## 安装命令
`pecl install swoole`

## 问题1:
`/private/tmp/pear/temp/swoole/include/swoole.h:492:10: fatal error: 'openssl/ssl.h' file not found`

google 了一圈，找到了

`ln -s /usr/local/opt/openssl/include/openssl /usr/local/include/`

大概意思就是创建个软连接到指定目录下。如果不知道自己的 openssl 目录，请用 `brew info openssl` 查看。

## 问题2:

尽管方式是解决了找不到 openssl 的问题，但是紧接着又遇到
`#error "Enable openssl support, require openssl library."`

又是一顿 google，最终找到

我们在运行 `brew info openssl` 时，下面会又一段：

``` 
For compilers to find openssl you may need to set:
export LDFLAGS="-L/usr/local/opt/openssl/lib"
export CPPFLAGS="-I/usr/local/opt/openssl/include"
```

只要在编译之前运行两个 export 命令即可解决掉 问题1和问题2。


## 安装结果

```
Build process completed successfully
Installing '/usr/local/Cellar/php@7.2/7.2.16/include/php/ext/swoole/config.h'
Installing '/usr/local/Cellar/php@7.2/7.2.16/pecl/20170718/swoole.so'
install ok: channel://pecl.php.net/swoole-4.3.1
Extension swoole enabled in php.ini
```