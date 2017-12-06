## python3 环境 flask+gunicorn+supervisor+nginx 在ubuntu 16.04 的部署

为了搭建我的个人网站：  [icser.me](https://www.icser.me)，我选择了flask这一轻量级python web框架。在搭建服务器的过程中，网上资料很多都有误，而且鉴于supervisor对于python3不支持的现状，经过探索，整理部署流程如下。

#### flask部署

flask的文档具体可以参见[documentation](http://flask.pocoo.org/docs/0.12/)，其中的描述已经很详尽。启动这样的一个flask项目，我们只需配置：

```python
# myproject/manager.py
# import ...
from flask import Flask # ...
app = Flask(__name__)

# ...

@app.route('/')
def main_page():
  return render_template("index.html")

if __name__ == '__main__':
  app.run(host='0.0.0.0', port='8000', debug=True)

```

**请记住上面的 app 为我们的应用，在之后的代码中我们仍然需要调用它。**

**同时，需要提示，我们需要以非 root 用户建立此项目， 防止用户攻入服务器，在这里，我的用户为web:web**

 `app.run(host='0.0.0.0', port='8000', debug=True)`

这行代码，定义了：向公网公开，端口为8000，启用debug模式（修改 manager.py 后，我们不需要重新启动进程）。我们只需运行

```powershell
$ python3 manager.py
 * Running on http://0.0.0.0:8000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 892-670-743

```

访问 [http://localhost:8000](http://localhost:8000) 即可看到我们的网页。

#### gunicorn

上面的flask内置服务器并不是一个理想的运行状态，它很不稳定，我们需要使用其他套件使它能够稳定存在。

gunicorn 代替了上面的 *app.run()* 代码，它实现了WSGI，能够处理 web 应用框架和 http 服务器之间的通信。

通过 pip3 安装gunicorn

`sudo pip3 install gunicorn`

我们只需运行：

`gunicorn -b 127.0.0.1:8000 -w 4 manager:app`

这行代码，我们运行了 manager.py 中的 app 应用，并将其绑定在了本地端口8000上。

需要注意，如果 manager.py 发生了变化，我们需要重新启动 gunicorn。

#### supervisor

上面的 gunicorn 虽然已经实现了通信，但是我们一旦关掉 terminal ，进程就会结束，我们需要把它作为 service 稳定运行，并且，一旦 gunicorn 意外结束进程，我们需要用 supervisor 及时地重启 gunicorn。

在网上的其他教程中，使用 python2 的 pip 直接安装 supervisor，但是由于 supervisor 仍未支持 python3，我们采用

`sudo apt-get install supervisor`

而 supervisor 的配置文件位于 */etc/supervisor/* 中，在 */etc/supervisor/supervisord.conf* 中，我们可以看到：

```
port=127.0.0.1:8000, 127.0.0.1:4000

; ...
[include]
files = /etc/supervisor/conf.d/*.conf
```

其中，port 规定了开放的端口：本地端口8000和4000。

[include]中为引用的文件，我们在 *conf.d/* 中创建 *project.conf* 来编辑我们的配置文件。

*project.conf*

```
 [program:project]
 directory = /path/to/project/
 command = gunicorn -b 127.0.0.1:8000 -w 4 manager:app
 autostart = true
 startsecs = 5
 autorestart = true
 startretries = 3
 user = web
 redirect_stderr = true
 stdout_logfile_maxbytes = 20MB
 stdout_logfile_backups = 20
 stdout_logfile = /path/to/project/logs/supervisor.log
```

  在这里，我们需要关注：

directory 为项目的绝对路径，command 即为我们在上边提到的 gunicorn 指令，注意，这里的port **一定** 要和 supervisord.conf 中的port 一致。 autorestart 和 autostart 即自动启动和自动重启。**user = web**，即为flask 项目的所有者。剩下的即为log文件配置等，注意，需要在项目文件夹下创建logs 文件夹下。

通过指令启动

`supervisord -c /etc/supervisor/supervisord.conf`

关于 supervisor 的指令

```powershell
$ supervisorctl reread      # 重新加载配置文件
$ supervisorctl update
$ supervisorctl start project
$ supervisorctl stop project
$ supervisorctl restart project
$ supervisorctl status project
$ supervisorctl help        # 查看更多命令
```

通过	命令`supervisorctl`即可进入supervisor 命令行。

如果我们改变了项目配置文件，我们需要reread > update （只重启改动的项目），或者reload （重启所有项目）。

如果这样程序仍未按照我们所想的运行，尝试 

`service supervisor restart`

即可实现配置文件的更新。

#### nginx

首先我们需要安装相应软件：

`sudo apt-get install nginx apache-utils`

为了搭建https 服务器，我们准备好 .crt 和.key 文件，并将其放在 */etc/nginx/conf.d/* 下。

在ubuntu16.04 环境下，nginx 的配置文件位置位于：*/etc/nginx/sites-enable/* 下

执行以下指令：

```powershell
$ cd /etc/nginx/sites-enable/
$ sudo cp default default.bak  # 备份default
$ sudo vim default # 编辑配置文件
```

修改配置文件：

```
 server {
     listen 80;
     server_name example.com www.example.com;
     rewrite ^/(.*) https://www.example.com/$1 permanent;
 }
 
 server {
     listen 443;
     listen [::]:443 ssl ipv6only=on;
     server_name www.example.com;
     if ( $host != 'www.example.com') {
         rewrite ^/(.*) https://www.example.com/$1 permanent;
     }
     
     root /path/to/project/;
     
     ssl on;
     ssl_certificate /etc/nginx/conf.d/1_www.example.com_bundle.crt;
     ssl_certificate_key /etc/nginx/conf.d/2_www.example.com.key;
     ssl_session_timeout 5m;
     ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
     ssl_prefer_server_ciphers on;
     
     index index.html;
     location /static/ {
         autoindex on;
     }
     location / {
         proxy_pass http://127.0.0.1:8000;
         proxy_set_header Host $http_host;
         proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
     }
     
     access_log /path/to/project//logs/access.log;
     error_log /path/to/project/logs/error.log;
 }
```

通过以上的配置文件，我们首先将80端口重定向至443端口，而后对配置 ssl证书，只需将认证证书包含其中即可。

之后，我们设置root 文件夹，便于定位flask 的static文件夹。

location /static/ 设置了 https://www.example.com/static/ 的配置。

location / 则设置了 https://www.example.com/ 根目录。我们将本地8000端口，映射到 / 根目录，并设置proxy_header。

最后，我们配置项目的log 文件位置即可。

此时，我们通过指令

`$ sudo service nginx restart`

或

`$ sudo nginx -s reload`

重载。访问https://www.example.com ，即可看到我们的网站。

