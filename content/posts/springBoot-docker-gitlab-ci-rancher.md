---
title: "springBoot+docker+gitlab-ci+rancher"
date: 2018-09-17T10:43:30+08:00
categories: ["tech"] 
tags: ["java"] 
draft: false
---


# SpringBoot+Docker+Gitlab-ci+Rancher微服务架构

> 最近在开发学习过程中遇到了微服务架构，采用的了一种自己认为的很牛逼的方式，我觉得很潮好吧，踩过一些坑，做了一下记录。

## 技术栈

- SpringBoot-Restful-Api
- Docker-容器
- Gitlab-ci-自动集成部署
- Ranchar-微服务容器

## 定义解释

- SpringBoot: 快速的，敏捷业务代码开发的，配置少的 JAVA Web 框架
- Docker: 官网的第一句话：`Build,Manage and Secure Your Apps Anywhere. Your Way.`简单的说就是一个提供给开发者的容器，类似 Tomcat 这样。
- GitLab-ci: 概念一：GitLab 是代码托管及版本控制工具；概念二：CI/CD 是 GitLab 的一部分，提供自动集成测试，流水线工作。
- Ranchar: `农场工人` 看翻译猜出-企业级容器管理平台,且开源。

## Step By Step

#### Step1 SpringBoot

- 创建一个 springboot project,支持 maven

- Maven 对 docker 的支持

  ```
             <plugin>
                  <groupId>com.spotify</groupId>
                  <artifactId>docker-maven-plugin</artifactId>
                  <version>0.4.14</version>
                  <configuration>
                      <imageName>你的生成的镜像名字</imageName>
                      <dockerDirectory>${project.basedir}</dockerDirectory>
                      <resources>
                          <resource>
                              <targetPath>/</targetPath>
                              <directory>${project.build.directory}</directory>
                              <include>${project.build.finalName}.jar</include>
                          </resource>
                      </resources>
                  </configuration>
                  <!-- 运行命令 mvn clean package docker:build 打包并生成docker镜像 -->
              </plugin>
  ```

- 根目录下创建 Dockerfile 文本文件

  ```
  FROM java:8
  EXPOSE 8080
  ARG JAR_FILE=xxx.jar # 你的 jar 包路径
  COPY ${JAR_FILE} app.jar
  ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","app.jar"]
  ```

- 此时你的 project 已经可以打包镜像了

#### Step2 Docker 安装、注册、配置

- Docker 分为两个版本，付费和免费，当然是先学习免费版啦。
- [官网下载对应的版本](https://docs.docker.com/) -> 安装 -> 注册 - > 学习命令
- 回 step1 的项目路径下 -> `mvn clean package docker:build` -> 查看本地的镜像文件(docker images) -> docker run -p 8080:8080 imageName -d (自行学习命令)
- 现在你的 docker 已经跑起来了，你的项目 

#### Step3 GitLab-ci

- [官网](https://gitlab.com/) -> 注册 -> 下载一个 git 版本控制工具 -> 配置自己的账户及 ssh -> 配置 gitlab ssh。（按步骤进行，如果使用过 github 则很简单）（至于为什么用 gitlab 源于它有 ci 集成，而 github 使用 Travis CI 等外部工具）

- new 一个 project

- 使用 docker 配置 Runner.

  ```
  # docker 拉取 runner 镜像
  sudo docker pull gitlab/gitlab-runner:latest
  # docker 添加容器 
  sudo docker run -d --name gitlab-runner --restart always \
    -v /srv/gitlab-runner/config:/etc/gitlab-runner \
    -v /var/run/docker.sock:/var/run/docker.sock \
    gitlab/gitlab-runner:latest
  # 注册 runner
  sudo docker exec -it gitlab-runner gitlab-ci-multi-runner register
  # 以下为注册需要的信息填写
  # gitlab-runner register
  Please enter the gitlab-ci coordinator URL:
  # 去到你的 gitlab project 下左边栏的 settings -> CI/CD -> Runners Settings 下面包含的你的 URL、token. 
  Please enter the gitlab-ci token for this runner:
  # xxxxxx
  # 其他信息自己对应填写即可(脚本建议 shell 或者 docker)
  
  
  # 启动 runner 即可看到你的 CI 下的 runner
  ```

#### step4 配置 Rancher

- [官网教程](https://www.cnrancher.com/quick-start/)
- `要求` 准备一台64位Linux主机(推荐Ubuntu 16.04)，至少4GB内存。在主机上安装支持的Docker版本，目前支持1.126、1.131或17.03.2三个版本。要在服务器上安装Docker，请遵循Docker的说明。
- $sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
- Rancher Server容器启动很快速，不到一分钟你就可以通过`https://<server_ip>`访问Rancher UI。
  一旦Rancher Server成功安装，用户界面将指导你添加第一个集群

#### step5 配置流水线

- 本地代码 -> push 到 gitlab project -> 执行 runner -> 集成并发布到 rancher 容器 -> rancher 启动镜像

- step1 项目根目录下新建 .gitlab-ci.yml （配置流水线操作步骤（类似脚本））

- [官方文档](https://docs.gitlab.com/ee/ci/yaml/)

- 贴出我的配置

  ```
  # 步骤
  stages:
  - build
  - builddocker
  - deploy
  
  # mvn 打包
  maven-build:
    image: maven:3-jdk-8
    stage: build
    cache:
      untracked: true
      key: ${CI_COMMIT_REF_SLUG}
      policy: push
    script:
    - mvn clean package
    - cp target/*.jar app.jar
    only:
    - master
  
  # docker 生成镜像
  job_build_docker_stable:
    stage: builddocker
    image: gitlab/dind
    cache:
      untracked: true
      key: ${CI_COMMIT_REF_SLUG}
      policy: pull
    only:
    - master
    script:
    - docker build -t wordmaster:stable .
  
  # 镜像发布到你的 rancher 
  job_deploy:
    stage: deploy
    image: registry.cn-hangzhou.aliyuncs.com/dev_tool/rancher-cli
    cache:
      untracked: true
      key: ${CI_COMMIT_REF_SLUG}
      policy: pull
    only:
    - master
    script:
    - rm -f ~/.rancher/cli.json
    - rancher --url (你的 rancher 的端点) --access-key (你的
    秘钥) --secret-key (秘钥 key) up -d  --pull --force-upgrade --conf
    irm-upgrade --stack wordmaster
  ```

- 到此完成。

## Rancher 的研究后期贴出

