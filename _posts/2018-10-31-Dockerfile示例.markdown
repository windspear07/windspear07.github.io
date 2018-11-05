---
layout: post
title:  "Dcokerfile示例"
date:   2018-10-31 12:07:01 +0000
categories: Linux Docker 
---

### 一、程序准备

app.py
      
        from flask import Flask
        app = Flask(__name__)

        @app.route("/")
        def hello():
        html = "name = {name} ; hostname= {hostname} :)"
        return html.format( name = os.getenv("NAME", "default"), hostname=socket.gethostname())
        
        if __name__ == "__main__"
        app.run(host='0.0.0.0', port=80)    

requirements.txt

        Flask

### 二、Dockerfile
        
        FROM python:2.7-slim

        # 将工作目录切换为 /app
        WORKDIR /app

        # 将当前目录下的所有内容复制到 /app 下
        ADD . /app

        # 使用 pip 命令安装这个应用所需要的依赖
        RUN pip install --trusted-host pypi.python.org -r requirements.txt

        # 允许外界访问容器的 80 端口
        EXPOSE 80

        # 设置环境变量
        ENV NAME World1

        # 设置容器进程为：python app.py，即：这个 Python 应用的启动命令
        CMD ["python", "app.py"]

### 三、打包镜像及运行

        docker build -t hh .
        docker run -idt -p 4000:80 hh

### 四、Dockerfile

[Dockerfile reference](https://docs.docker.com/engine/reference/builder/#usage)