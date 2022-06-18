
`docker logs` 是我们经常用来查看容器运行日志的命令，但是在长时间容器运行后，会产生大量的日志，会发现越来越慢，所以我们需要清理日志。

## Docker 清理日志

通过命令 `docker inspect --format='{{.LogPath}}' <容器ID>` 查看容器的日志路径

通过命令 `echo > <日志路径>` 或者 `cat /dev/null > <日志路径>` 清空容器的日志

## Docker 限制日志大小

新建或修改 `/etc/docker/daemon.json` 文件, 内容如下：
```json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "500m",   // 日志文件最大大小
        "max-file": "3"       // 日志文件最大数量
    }
}
```

然后重启docker的守护线程
```bash
systemctl daemon-reload
systemctl restart docker
```
