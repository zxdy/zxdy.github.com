今天openvpn貌似有点抽风， 不好使了。。。sigh。。
听说shadowsocks挺好用，就尝试了一下，确实还挺方便。记录一下搭建过程，基本上照官方的user manual来就ok了

1. 安装
--

```bash
apt-get install python-pip python-dev build-essential
python --version #安装之前检查一下vps的python版本号，需要2.6或2.7以上
pip install shadowsocks #安装
``` 

2. 配置文件
--

新建一个名为`config.json`的配置文件

```bash
{
          "server":"vps的ip",
          "server_port":8388,
          "local_port":1080,
          "password":"barfoo!", #认证密码
          "timeout":600,
          "method":"table" #加密方式，默认table，推荐aes-256-cfb
}
```

*如果想用除table以外的加密方式，需要额外安装M2Crypto*

```bash
apt-get install python-m2crypto
#或者
pip install M2Crypto
```

3.进入`config.json`所在目录，运行shadowsocks
--

```bash
nohup ssserver > shadowsocks_log &
```

4.客户端
--

下载[shadowsocks-gui][1]，填入相应的vps ip地址，端口，密码信息后保存  

5. 浏览器代理
--
新建socket5代理，地址填127.0.0.1，端口1080保存就可以使用了


  [1]: http://sourceforge.net/projects/shadowsocksgui/files/dist/