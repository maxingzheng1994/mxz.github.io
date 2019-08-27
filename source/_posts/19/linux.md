# linux
### 批量杀死进程
```
ps aux | grep nginx | cut -c 9-15| xargs kill -9

kubectl get pods -l com.qinsilk.app=ths |grep ths-| cut -c 1-20|xargs kubectl delete pods
```
1. ps aux 查看所有进程的命令
2. grep "common" 在前序命令查找到的进程中过滤出存在关键字 common 的进程
3. cut -c 9-15 在前序命令过滤出的进程中截取输出行的第 9 到 15 个字符，正好是进程 PID
4. xargs 将前序命令得到的结果作为 kill -9 的参数

### 开机启动
修改~/.bashrc


### 任务后台运行
```
./nginx_reload.sh &
```
& 将命令 放入到作业队列中 ` jobs -l` 查看

### 定时任务

### 循环任务
小技巧
```
#!/bin/bash
step=3
for((i=0;;i=(i+step)));do
    $(nginx -s reload)
    sleep $step
done
exit 0
```
### 定时监控

```
watch -d -n 1 '命令'
```

### 查找命令

#### locate
#### find
> find /home/ -name "*.txt"
#### grep
> grep -l  列出查找文件名  -on  显示搜索的关键字和行号 

### 解压缩指令 tar
#### 打包
> tar -czvf test.tar /etc/host*      将etc下以host开头的文件打包为test.tar
> tar -zxvf test.tar
### 配置文件
> vim ~/.profile
> source ~/.profile
### 管道符 |

### 拷贝传输文件
scp -r  目标  位置

### 显示磁盘大小
df -lh 

### vim操作
vim /要查找的单词   n 向下继续查找
:%s/word1/word2/g   将Word1 替换为word2

### 别名alias  s=''  unalias
```
vi /etc/bashrc   source /etc/bashrc
alias svim='sudo vim'
alias c7='chmod 777'
alias cx='chmod +x'
alias ..="cd .."
alias ...="cd ..; cd .."
```

### 快捷键
ctrl+u   删除一整行
ctrl+w 向前删除一个单词
alt+b alt+f  左右以单词为单位移动
ctrl+a ctrl+e  移至行首或行尾
ctrl+k  删除到行尾

pstree -p

### 对比命令diff
通过使用 <(some command) 可以将输出视为文件。例如，对比本地文件 /etc/hosts 和一个远程文件
```
diff /etc/hosts <(ssh somehost cat /etc/hosts)
```

### tee
输出到文件
> ls -al | tee file.txt