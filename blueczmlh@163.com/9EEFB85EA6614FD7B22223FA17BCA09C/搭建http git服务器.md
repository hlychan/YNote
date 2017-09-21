# <center> HTTP GIT服务器的搭建 </center>
 
### 1. apache2,git-core,gitweb安装：

```
sudo apt-get install apache2 git-core gitweb
```
    
### 2. 创建git库

选择/home/git-http为仓库存储路径

    cd /home/git-http
    git init --bare test.git # 创建一个空的仓库
    chwon -R www-data:www-data test.git # 使apache2对仓库有访问权限
    
修改test.git/description添加对仓库的描述

打开/home/git-http/test.git/config，添加如下内容，否则push时会提示403错误：

        [http]
            receivepack = true

### 3. apache2配置：

修改/etc/apache2/httpd.conf为如下内容：

    ServerName 10.141.199.18:80
    # 默认apache2只对两个目录（/usr/share和/var/www）有访问权限，如下指令赋予apache2对/srv的访问权限。
    <Directory /home/git-http/>
        Options Indexes FollowSymLinks
        AllowOverride None
        #Require all granted # 不注释掉连不上服务器
    </Directory>
    # 如果没有这句，在其他机器上执行git clone等命令会返回403错误，参照最后一条“参考”
    <Location />
        Options +ExecCGI
        #Require all granted # 不注释掉连不上服务器
    </Location>
    # 设置git的工程目录
    SetEnv GIT_PROJECT_ROOT /home/git-http/
    # 默认情况下，含有git-daemon-export-ok文件的目录才可以被导出（用作git库目录）。设置这个环境变量以便所有目录都可以被导出
    SetEnv GIT_HTTP_EXPORT_ALL
    # 虚拟主机，匹配80端口的任何ip地址的请求，访问gitweb
    <virtualhost *:80>
        # 顺便在/etc/hosts里添加上一句：127.0.0.1 git.server.com。这样，在服务器上可以通过该名字访问这个页面
        ServerName git.server.com
        DocumentRoot /usr/share/gitweb
        ErrorLog ${APACHE_LOG_DIR}/git_error.log
        CustomLog ${APACHE_LOG_DIR}/git_access.log combined
    </virtualhost>
    # gitweb目录添加ExecCGI的功能
    <Directory /usr/share/gitweb>
        Options FollowSymLinks ExecCGI
        AddHandler cgi-script .cgi
        DirectoryIndex gitweb.cgi
    </Directory>
    # 对git库的各种请求，执行git-http-backend.cgi
    ScriptAliasMatch \
        "(?x)^/(.*/(HEAD | \
        info/refs | \
        objects/(info/[^/]+ | \
        [0-9a-f]{2}/[0-9a-f]{38} | \
        pack/pack-[0-9a-f]{40}\.(pack|idx)) | \
        git-(upload|receive)-pack))$" \
        /usr/lib/git-core/git-http-backend/$1
    # 其余的请求，执行gitweb.cgi
    ScriptAlias / /usr/share/gitweb/gitweb.cgi
    # 设置git push等操作的认证方式为文件认证，/var/www/git-auth后面会创建
    <LocationMatch "^/.*/git-receive-pack$">
        AuthType Basic
        AuthName "Git Access"
        Require valid-user
        AuthBasicProvider file
        AuthUserfile /var/www/git-auth
    </LocationMatch>

### 4、push操作的认证

默认git-http-backend的upload-pack是被置为真的，即可以执行git clone/pull/fetch。但是，默认receive-pack是被置为false，即不能git push。为了支持带认证的git push，需要两步操作。

第一步，打开/home/git-http/test.git/config，添加如下内容：

    [http]
        receivepack = true
        
如果不加上面这句，git clone下来的版本库，git push时会提示403错误，即没有授权

第二步，生成一个包含用户名和密码的文件，该文件能被apache2读取，作为文件认证的依据：

    htpasswd -c /var/www/git-auth jacky # 命令意思为在/var/www下生成git-auth认证文件，添加jacky用户。注意"-c"选项只在第一次生成认证文件时需要

### 5、gitweb的配置

修改/etc/gitweb.conf中的一句：

    $projectroot = "/home/git-http"

### 6、重启apache2

    sudo service apache2 restart

### 7、客户端检查

在客户端电脑上，找一个目录，执行如下命令

    git clone http://10.141.199.18/test.git
    cd test
    echo "test" > test.txt
    git add test.txt
    git commit -s  #在弹出的文本编辑器中输入注释
    git push origin master
然后，在浏览器输入http://10.141.199.18，查看刚才的操作是否记录到gitweb上了