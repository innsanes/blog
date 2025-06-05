## MySQL

```bash
docker run -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=blog -d mysql:8.0.42
```
因为我觉得docker的自动命名还挺酷的，同时我也并没有启动两个相同image的需求，所以就没有设置name，如果需要的话可以设置一个自己认识的名字
```bash
docker run --name mysql8 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=blog -d mysql:8.0.42
```
设置密码是必须的，也就是通过环境变量参数传入，否则就启动不起来，会打印出提示信息：
```
You need to specify one of the following as an environment variable:
- MYSQL_ROOT_PASSWORD
- MYSQL_ALLOW_EMPTY_PASSWORD
- MYSQL_RANDOM_ROOT_PASSWORD
```

## Redis

参数d是必须的，这代表detached，我们执行完这一行命令就可以让这个container在后台运行，我们转而去做其他事情。
```bash
docker run -d redis:8.0
```
