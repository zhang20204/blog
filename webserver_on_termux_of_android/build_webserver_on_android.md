[toc]

# 在安卓机上基于termux搭建web服务器
# 简介
换了新手机后，旧手机闲置着总觉得浪费，所以就想着折腾一下，搭建一个简单的web服务器玩玩。

# 搭建详情
## 安装应用
* 需要安装的应用：
    * apache: 服务器软件,用来搭建本地web服务
    * termux-services: 管理termux服务
    * cpolar: 内网穿透,是本地服务公网也可以访问
* 安装apache,termux-services
```shell
pkg install apache2 termux-services
```
* 安装 cpolar
```shell
# 创建必要的文件夹：
mkdir -p $PREFIX/etc/apt/sources.list.d

# 添加cpolar下载源
echo "deb [trusted=yes] http://termux.cpolar.com termux extras" >> $PREFIX/etc/apt/sources.list.d/cpolar.list

# 更新仓库
pkg update

# 安装
pkg install cpolar
```
安装完成后重启 termux
## 注册 cpolar 账户
* 注册  
登陆[cpolar官网](https://www.cpolar.com/)，点击 **免费注册**，填写必要的信息进行注册(注册时需要的信息可以随便填写，但是，如果填写邮箱时如果填写的是不存在的邮箱，虽然可以注册成功，但后期将无法更换密码，因为她需要邮箱进行验证)
* 获取 authtoken  
登陆成功后便可以获取 **authtoken**(cpolar软件配置内网穿透需要这个)，点击 **验证** 即可找到 authtoken。
## 配置&启动
* 配置
    * apache2  
        * 打开：$PREFIX/etc/apache2/httpd.conf
        * 取消 `ServerName www.example.com` 的注释
        * 将 `www.example.com` 更改为本地地址和自己向要配置的端口号，例如:`127.0.0.1:8080`
    * cpolar
        * 打开：$PREFIX/etc/cpolar/cpolar.yml
        * 添加 `authtoken: <your authtoken>`
* 启动服务
```shell
# 启动 apache2
apachectl start

# 启动 cpolar
sv up cpolar
```
* cpolar 穿透 apache 本地服务
```shell
# 这里的ip和端口就是 apache2 配置文件中配置的ip和端口号
cpolar http 127.0.0.1:8080
```
* 查看本地web服务映射的URL  
登陆cpolar 网站，点击 **状态** 可以查看 **在线隧道** 信息，其中包括本地服务映射的 **URL** ，复制该 URL 并在浏览器中打开就可以访问 web 服务了。
