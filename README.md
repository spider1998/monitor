# monitor
# 基于Prometheus+Grafana+docker-compose的可视化服务监控搭建

利用docker-compose编排所需镜像，映射关联端口，并在prometheus下的prometheus.yml中配置job,最终在docker-compose.yml目录下一键启动：

1.安装docker

	$ curl -fsSL https://get.docker.com -o get-docker.sh

	$ sudo sh get-docker.sh

	免sudo权限
	sudo groupadd docker

	sudo gpasswd -a ${USER} docker

	sudo service docker restart

	newgrp - docker

	测试：docker run hello-world

2.安装docker-compose

	$ sudo curl -L https://github.com/docker/compose/releases/download/1.24.0-rc1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

	$ sudo chmod +x /usr/local/bin/docker-compose（授权）

	$ docker-compose --version（验证版本）

	解决镜像垃取慢问题：
		在 /etc/docker/daemon.json 文件中添加以下参数（没有该文件则新建）：
		{
		  "registry-mirrors": ["https://9cpn8tt6.mirror.aliyuncs.com"]
		}
		服务重启：
		systemctl daemon-reload
		systemctl restart docker

3.部署prometheus

	撰写docker-compose.yml(挂载时可挂载整个目录)
  ```yml
  prometheus:
        container_name: prometheus
        image: prom/prometheus
        restart: always
        volumes:
          - ~/pro/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
        command:
          - '--config.file=/etc/prometheus/prometheus.yml'
        ports:
          - '9090:9090'
  ```
  
	撰写prometheus.yml
  ```yml
  global:
  scrape_interval:     5s
  scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets:
      - 'prometheus:9090'
  ```
	docker-compose up -d
	
//targets中可直接用容器名：端口号对应服务

4. 使用node exporter收集os监控数据
	
	docker-compose.yml 添加配置，垃取node-exporter服务镜像,并指定端口
  ```yml
   node-exporter:
        container_name: node-exporter
        image: prom/node-exporter
        ports:
          - '9100:9100'
  ```
	prometheus添加job
  ```yml
  - job_name: 'node'
    static_configs:
    - targets:
      - 'node-exporter:9100'
  ```

5. 使用mysqld-exporter收集mysql监控数据

  	docker-compose.yml 添加配置，拉取mysqld-exporter服务镜像,并指定端口
    ```yml
    mysql-exporter:
        image: prom/mysqld-exporter
        ports:
          - '9104:9104'
        environment:
          - DATA_SOURCE_NAME=root:1223456@(localhost:3306)/
        restart: always
    ```
     使用${DATA_SOURCE_NAME}指定数据库配置
    prometheus添加job
    ```yml
     - job_name: 'mysql'
      static_configs:
      - targets:
        - 'mysql-exporter:9104'
    ```
6. 使用cadVisor收集docker监控数据 
  docker-compose.yml 添加配置，拉取cadvisor服务镜像,并指定端口
  ```yml
  cadvisor:
      image: google/cadvisor:latest
      ports:
        - '8080:8080'
      volumes:
        - /:/rootfs:ro
        - /var/run:/var/run:rw
        - /sys:/sys:ro
        - /var/lib/docker/:/var/ib/docker:ro
        - /dev/disk/:/dev/disk:ro
      restart: unless-stopped
  ```
 prometheus添加job
 ```yml
 - job_name: 'cadvisor'
    scrape_interval: 5s
    static_configs:
      - targets:
        - 'cadvisor:8080'
 ```
 至此，所有数据源服务已经提供，服务启动后可通过浏览器访问对应端口可查看结果，可在localhost:9090/targets下看所有job状态
 接下来将数据进行可视化展示
7.安装配置grafana
	docker-compose.yml 添加配置，垃取grafana服务镜像,并指定端口
  ```yml
  grafana:
        image: grafana/grafana
        volumes:
          - ~/pro/grafana_data:/varlib/grafana
        environment:
          - GF_SECURITY_ADMIN_PASSWORD=pass
        depends_on:
          - prometheus
        ports:
          - '3000:3000'
  ```
  服务启动后浏览器访问localhost:3000可跳转到登录页面，用户名admin密码为${GF_SECURITY_ADMIN_PASSWORD}，未指定则默认为admin
  
  配置grafana:
  
  1. create datasource :name自定义，type选prometheus,url填写http://localhost:9090; access选browser;保存并测试
  
  2.导入模板，可自定义添加画板或导入官方模板https://grafana.com/grafana/dashboards（填入id即可）
  
    //本次测试模板：os:8919  mysql:11323 docker:179 
    
  3.展示数据，完成部署。
  
8.最终效果

os监控

![](https://github.com/spider1998/monitor/blob/master/os.jpg)

mysql监控

![](https://github.com/spider1998/monitor/blob/master/mysql.jpg)

docker监控

![](https://github.com/spider1998/monitor/blob/master/docker.jpg)
  
  
  
  
  
  
  
