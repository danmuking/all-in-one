1. 打开一个终端窗口。
2. 下载 jenkinsci/blueocean 镜像并使用以下docker run 命令将其作为Docker中的容器运行 ：
```
docker run -d -p 18727:8080 -v /home/ubuntu/jenkins/:/var/jenkins_home -v /etc/localtime:/etc/localtime -v /var/run/docker.sock:/var/run/docker.sock --name jenkins jenkins/jenkins
```

