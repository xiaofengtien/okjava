###  进入docker容器：

```bash
docker  exec -it yunpei-sxyp-admin /bin/sh
```

### 打印dump日志：

```bash
jstack -f 5824

jmap -dump:format=b,file=文件名 [pid] 利用Jmap dump
```

### 查看某个时间段的日志

```bash 
docker logs --since="2021-01-16T09:47:00" --until "2021-01-16T09:50:00" containerID 
```

### 查看某个时间点后面的日志

```bash
 docker logs --since="2021-01-28T09:30:00"  8dee6f4c307b
```

### 查看实时日志

```bash
docker logs -f containerID 
```

### 进入容器

```bash
docker exec -it containerID sh || docker exec -it containerID /bin/sh
```

### 查看100条实时日志

```bash
docker logs -f -t --tail 100
```

### ???

```bash
docker inspect   mdm-dev
```

