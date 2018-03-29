# repo
repository for server


### elk 
docker-compose for elk

如果单独启动则需要通过 docker network connect 命令吧比如kibana加入到elasticsearch的网络中，
否则kibana不能通过docker内置的dns解析到elasticsearch的ip，从而会连不上。

### k8s
1. minikube 
