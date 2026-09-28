# 安装TUN

```shell
apk update
apk add kmod-tun
modprobe tun
```

# 测试配置

```shell
cd /app/mihomo
./mihomo -d /app/mihomo
```

# 启动服务

```shell
/etc/init.d/mihomo start
```