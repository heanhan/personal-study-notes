

### 查看ip地址

```sh
#内网 
osascript -e "IPv4 address of (system info)"

#公网
dig +short myip.opendns.com @resolver1.opendns.com

```

##### 查看 DNS 信息

```sh
cat /etc/resolv.conf | sed -n '16p' | awk '{print $2}' 

```

### 获取电脑的cpu信息

```sh
sysctl -n machdep.cpu.brand_string
```

### cpu物理核心数

```sh
sysctl -n hw.physicalcpu 
```

### cpu线程数

```sh
sysctl -n hw.logicalcpu
```

