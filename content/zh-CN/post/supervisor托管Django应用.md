---
title: "Supervisor托管Django应用"
date: 2021-12-19T03:33:29+08:00
draft: false
---

Supervisor 是用Python开发的一套通用的进程管理程序，能将一个普通的命令行进程变为后台daemon，并监控进程状态并且异常退出时能自动重启，也很适合部署celery。



# 环境

本文所演示的环境文centos，python2.7版本



# 安装

本文使用pip安装，也可使用yum

```shell
pip install supervisor复制代码
```

安装完毕后，查看版本如下：

```javascript
pip3 list | grep su
supervisor              4.0.4   
```

使用pip的安装可能没有设置好supervisor的环境变量，还需要查看一下supervisor安装后的二进制可执行文件在哪里。如果二进制文件本身就在/usr/bin下就说明已经安装好了可以跳过软链接。

- 搜索在 `/` 目录下，前后模糊查询名称为 `supervi` 的文件，如下：

```javascript
find / -name "*supervi*" -ls | grep python3 | grep bin
405327    4 -rwxr-xr-x   1 root     root          242 Oct 12 18:42 /usr/local/python3/bin/supervisorctl
405325    4 -rwxr-xr-x   1 root     root          237 Oct 12 18:42 /usr/local/python3/bin/echo_supervisord_conf
405328    4 -rwxr-xr-x   1 root     root          240 Oct 12 18:42 /usr/local/python3/bin/supervisord
```

- 将 `supervisorctl`、`echo_supervisord_conf` 和 `supervisord` 添加软链到执行目录下/usr/bin

```javascript
ln -s /usr/local/python3/bin/echo_supervisord_conf /usr/bin/echo_supervisord_conf
ln -s /usr/local/python3/bin/supervisord /usr/bin/supervisord
ln -s /usr/local/python3/bin/supervisorctl /usr/bin/supervisorctl
```

此时算是安装好了。



# 配置

- 我的做法是将主配置文件放在/etc/supervisord.conf中，应用配置文件放在/etc/supervisord.d/文件下。echo_supervisord_conf用于生成默认配置，我们生成默认配置并将最后两行引入子进程配置文件的注视去掉

  ```shell
  mkdir /etc/supervisord.d
  echo_supervisord_conf > /etc/supervisord.conf
  ```

  supervisord.conf中修改的内容

  ```shell
  [include]
  files = supervisord.d/*.ini
  ```

- 再来配置应用配置文件

  ```shell
  # 创建文件
  vim /etc/supervisord.d/demo.ini
  
  # 文件中配置的内容
  [program:demo]
  command=/root/pyenv/demo/bin/gunicorn config.wsgi -c config/gunicorn_conf.py
  directory=/data/web/demo/coding                ; directory to cwd to before exec (def no cwd)
  autostart=true                ; start at supervisord start (default: true)
  autorestart=true              ; retstart at unexpected quit (default: true)
  startsecs=10                  ; number of secs prog must stay running (def. 1)
  startretries=3                ; max # of serial start failures (default 3)
  stopwaitsecs=600               ; max num secs to wait b4 SIGKILL (default 10)
  stdout_logfile=/logs/demo.log        ; stdout log path, NONE for none; default AUTO
  ```

- 我使用的是gunicorn来部署应用，需要在应用根目录下配置gunicorn配置文件

  ```shell
  # 创建文件
  vim config/gunicorn_conf.py
  
  # 文件内容
  # -*- coding: utf-8 -*-
  import multiprocessing
  
  
  bind = "0.0.0.0:8100"
  workers = 3
  loglevel = "debug"
  
  worker_connections = 1000
  timeout = 50
  keepalive = 2
  ```

- 另外这里是我主配置文件的全部内容，仅供参考

  ```shell
  
  ; Sample supervisor config file.
  ;
  ; For more information on the config file, please see:
  ; http://supervisord.org/configuration.html
  ;
  ; Notes:
  ;  - Shell expansion ("~" or "$HOME") is not supported.  Environment
  ;    variables can be expanded using this syntax: "%(ENV_HOME)s".
  ;  - Quotes around values are not supported, except in the case of
  ;    the environment= options as shown below.
  ;  - Comments must have a leading space: "a=b ;comment" not "a=b;comment".
  ;  - Command will be truncated if it looks like a config file comment, e.g.
  ;    "command=bash -c 'foo ; bar'" will truncate to "command=bash -c 'foo ".
  ;
  ; Warning:
  ;  Paths throughout this example file use /tmp because it is available on most
  ;  systems.  You will likely need to change these to locations more appropriate
  ;  for your system.  Some systems periodically delete older files in /tmp.
  ;  Notably, if the socket file defined in the [unix_http_server] section below
  ;  is deleted, supervisorctl will be unable to connect to supervisord.
  
  [unix_http_server]
  file=/var/run/supervisor/supervisor.sock   ; the path to the socket file
  ;chmod=0700                 ; socket file mode (default 0700)
  ;chown=nobody:nogroup       ; socket file uid:gid owner
  ;username=user              ; default is no username (open server)
  ;password=123               ; default is no password (open server)
  
  ; Security Warning:
  ;  The inet HTTP server is not enabled by default.  The inet HTTP server is
  ;  enabled by uncommenting the [inet_http_server] section below.  The inet
  ;  HTTP server is intended for use within a trusted environment only.  It
  ;  should only be bound to localhost or only accessible from within an
  ;  isolated, trusted network.  The inet HTTP server does not support any
  ;  form of encryption.  The inet HTTP server does not use authentication
  ;  by default (see the username= and password= options to add authentication).
  ;  Never expose the inet HTTP server to the public internet.
  
  ;[inet_http_server]         ; inet (TCP) server disabled by default
  ;port=127.0.0.1:9001        ; ip_address:port specifier, *:port for all iface
  ;username=user              ; default is no username (open server)
  ;password=123               ; default is no password (open server)
  
  [supervisord]
  logfile=/var/log/supervisor/supervisord.log ; main log file; default $CWD/supervisord.log
  logfile_maxbytes=50MB        ; max main logfile bytes b4 rotation; default 50MB
  logfile_backups=10           ; # of main logfile backups; 0 means none, default 10
  loglevel=info                ; log level; default info; others: debug,warn,trace
  pidfile=/var/run/supervisord.pid ; supervisord pidfile; default supervisord.pid
  nodaemon=false               ; start in foreground if true; default false
  silent=false                 ; no logs to stdout if true; default false
  minfds=1024                  ; min. avail startup file descriptors; default 1024
  minprocs=200                 ; min. avail process descriptors;default 200
  ;umask=022                   ; process file creation umask; default 022
  ;user=supervisord            ; setuid to this UNIX account at startup; recommended if root
  ;identifier=supervisor       ; supervisord identifier, default is 'supervisor'
  ;directory=/tmp              ; default is not to cd during start
  ;nocleanup=true              ; don't clean up tempfiles at start; default false
  ;childlogdir=/tmp            ; 'AUTO' child log dir, default $TEMP
  ;environment=KEY="value"     ; key value pairs to add to environment
  ;strip_ansi=false            ; strip ansi escape codes in logs; def. false
  
  ; The rpcinterface:supervisor section must remain in the config file for
  ; RPC (supervisorctl/web interface) to work.  Additional interfaces may be
  ; added by defining them in separate [rpcinterface:x] sections.
  
  [rpcinterface:supervisor]
  supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
  
  ; The supervisorctl section configures how supervisorctl will connect to
  ; supervisord.  configure it match the settings in either the unix_http_server
  ; or inet_http_server section.
  
  [supervisorctl]
  serverurl=unix:///var/run/supervisor/supervisor.sock ; use a unix:// URL  for a unix socket
  ;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
  ;username=chris              ; should be same as in [*_http_server] if set
  ;password=123                ; should be same as in [*_http_server] if set
  ;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
  ;history_file=~/.sc_history  ; use readline history if available
  
  ; The sample program section below shows all possible program subsection values.
  ; Create one or more 'real' program: sections to be able to control them under
  ; supervisor.
  
  ;[program:theprogramname]
  ;command=/bin/cat              ; the program (relative uses PATH, can take args)
  ;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
  ;numprocs=1                    ; number of processes copies to start (def 1)
  ;directory=/tmp                ; directory to cwd to before exec (def no cwd)
  ;umask=022                     ; umask for process (default None)
  ;priority=999                  ; the relative start priority (default 999)
  ;autostart=true                ; start at supervisord start (default: true)
  ;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
  ;startretries=3                ; max # of serial start failures when starting (default 3)
  ;autorestart=unexpected        ; when to restart if exited after running (def: unexpected)
  ;exitcodes=0                   ; 'expected' exit codes used with autorestart (default 0)
  ;stopsignal=QUIT               ; signal used to kill process (default TERM)
  ;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
  ;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
  ;killasgroup=false             ; SIGKILL the UNIX process group (def false)
  ;user=chrism                   ; setuid to this UNIX account to run the program
  ;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
  ;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
  ;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
  ;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
  ;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
  ;stdout_events_enabled=false   ; emit events on stdout writes (default false)
  ;stdout_syslog=false           ; send stdout to syslog with process name (default false)
  ;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
  ;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
  ;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
  ;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
  ;stderr_events_enabled=false   ; emit events on stderr writes (default false)
  ;stderr_syslog=false           ; send stderr to syslog with process name (default false)
  ;environment=A="1",B="2"       ; process environment additions (def no adds)
  ;serverurl=AUTO                ; override serverurl computation (childutils)
  
  ; The sample eventlistener section below shows all possible eventlistener
  ; subsection values.  Create one or more 'real' eventlistener: sections to be
  ; able to handle event notifications sent by supervisord.
  
  ;[eventlistener:theeventlistenername]
  ;command=/bin/eventlistener    ; the program (relative uses PATH, can take args)
  ;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
  ;numprocs=1                    ; number of processes copies to start (def 1)
  ;events=EVENT                  ; event notif. types to subscribe to (req'd)
  ;buffer_size=10                ; event buffer queue size (default 10)
  ;directory=/tmp                ; directory to cwd to before exec (def no cwd)
  ;umask=022                     ; umask for process (default None)
  ;priority=-1                   ; the relative start priority (default -1)
  ;autostart=true                ; start at supervisord start (default: true)
  ;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
  ;startretries=3                ; max # of serial start failures when starting (default 3)
  ;autorestart=unexpected        ; autorestart if exited after running (def: unexpected)
  ;exitcodes=0                   ; 'expected' exit codes used with autorestart (default 0)
  ;stopsignal=QUIT               ; signal used to kill process (default TERM)
  ;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
  ;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
  ;killasgroup=false             ; SIGKILL the UNIX process group (def false)
  ;user=chrism                   ; setuid to this UNIX account to run the program
  ;redirect_stderr=false         ; redirect_stderr=true is not allowed for eventlisteners
  ;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
  ;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
  ;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
  ;stdout_events_enabled=false   ; emit events on stdout writes (default false)
  ;stdout_syslog=false           ; send stdout to syslog with process name (default false)
  ;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
  ;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
  ;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
  ;stderr_events_enabled=false   ; emit events on stderr writes (default false)
  ;stderr_syslog=false           ; send stderr to syslog with process name (default false)
  ;environment=A="1",B="2"       ; process environment additions
  ;serverurl=AUTO                ; override serverurl computation (childutils)
  
  ; The sample group section below shows all possible group values.  Create one
  ; or more 'real' group: sections to create "heterogeneous" process groups.
  
  ;[group:thegroupname]
  ;programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
  ;priority=999                  ; the relative start priority (default 999)
  
  ; The [include] section can just contain the "files" setting.  This
  ; setting can list multiple files (separated by whitespace or
  ; newlines).  It can also contain wildcards.  The filenames are
  ; interpreted as relative to this file.  Included files *cannot*
  ; include files themselves.
  
  [include]
  files = supervisord.d/*.ini                                                                                                
  ```



# Supervisor

- 启动supervisor

  ```shell
  $ supervisord
  
  # 有时候启动会报错需要unlink sock 文件
  unlink /var/run/supervisor/supervisor.sock
  ```

  即可运行。

  > supervisor 默认在以下路径查找配置文件：/usr/etc/supervisord.conf, /usr/supervisord.conf, supervisord.conf, etc/supervisord.conf, /etc/supervisord.conf, /etc/supervisor/supervisord.conf

  如需指定主配置文件，则需要使用`-c`参数：

  ```
  $ supervisord -c /etc/supervisord.conf
  ```

- 关闭

  ```shell
  supervisorctl shutdown
  ```

# Supervisorctl

- 启动完supervusior后

  ```shell
  $ supervisorctl reread
  $ supervisorctl update
  ```

  这两个命令分别代表重新读取配置、更新子进程组。执行update后输出：

  ```shell
  demo: added process group
  ```

- 然后启动应用，就完成了！

  ```shell
  $ supervisorctl strat demo
  ```

- 以下是supervisorctl的介绍和相关命令

  supervisorctl 是客户端程序，用于向supervisord发起命令。

  通过`supervisorctl -h`可以查看帮助说明。我们主要关心的是其`action`命令：

  ```
  $ supervisorctl  help
  
  default commands (type help <topic>):
  =====================================
  add    exit      open  reload  restart   start   tail   
  avail  fg        pid   remove  shutdown  status  update 
  clear  maintail  quit  reread  signal    stop    version复制代码
  ```

  这些命令对于控制子进程非常重要。示例:

  ```
  reread ;重新加载配置文件
  update ;将配置文件里新增的子进程加入进程组，如果设置了autostart=true则会启动新新增的子进程
  status ;查看所有进程状态
  status <name> ;查看指定进程状态
  start all; 启动所有子进程
  start <name>; 启动指定子进程
  restart all; 重启所有子进程
  restart <name>; 重启指定子进程
  stop all; 停止所有子进程
  stop <name>; 停止指定子进程
  reload ;重启supervisord
  add <name>; 添加子进程到进程组
  reomve <name>; 从进程组移除子进程，需要先stop。注意：移除后，需要使用reread和update才能重新运行该进程复制代码
  ```

  supervisord 有进程组(process group)的概念：只有子进程在进程组，才能被运行。

  `supervisorctl`也支持交互式命令行：

  ```
  $ supervisorctl
  demo                        RUNNING   pid 27188, uptime 0:05:09
  supervisor> version
  3.3.4
  supervisor> 
  ```



# 参考

https://juejin.cn/post/6844903745587773448
