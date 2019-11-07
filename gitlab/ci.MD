# 简介

![ci示意图](https://docs.gitlab.com/ee/ci/introduction/img/gitlab_workflow_example_11_9.png)
### CI
持续集成是指对于每次向存储库的推送，都可以创建一组脚本来自动构建和测试应用程序。对于提交给每个分支的每个更改，自动连续地构建和测试，以确保所引入的更改通过为应用程序建立的测试，准则和代码合规性标准，从而减少了向应用程序引入错误的机会。

### CD
持续交付是超越持续集成的一步。应用程序不仅会在推送到代码库的每次代码更改时都进行构建和测试，而且作为附加步骤，提供手动或者自动触发的部署步骤。

### 参考资料

[Gitlab CI 介绍](https://docs.gitlab.com/ee/ci/introduction/)

# 优点

提高效率：将测试、部署等重复性的工作交给工具去做，解放人力。

发现问题：持续集成自动获取变更，自动的测试与代码风格检查，更早的发现问题，解决问题。

快速交付：省去了手工运维上线的烦恼，同时避免了人工可能带来的失误，带来了更好的部署方式，更高的部署效率。更同时可以很方便的回退到任一版本，保证服务始终有一个可用的版本。

# 使用gitlab集成

GitLab CI 是GitLab内置的进行持续集成的工具，只需要在仓库根目录下创建 `.gitlab-ci.yml` 文件，并配置 `GitLab Runner` ，每次提交的时候，gitlab将自动识别到.gitlab-ci.yml文件，并且使用Gitlab Runner执行该脚本。

因此，简而言之，CI所需要的步骤可以归结为：
```
    1. 添加`.gitlab-ci.yml`到项目的根目录
    2. 配置一个Runner
```
从此刻开始，在每一次push到Git仓库的过程中，Runner会自动开启pipline，pipline将显示在项目的Pipline页面中。

# 配置 `.gitlab-ci.yaml`

`.gitlab-ci.yml` 是用来配置CI在我们的项目中做些什么工作。它位于项目的根目录。

在任何的push操作，GitLab都会寻找 `.gitlab-ci.yml` 文件，并对此次commit开始jobs，jobs的内容来源于 `.gitlab-ci.yml` 文件。

下面是一个 `Node.js` 项目的简单示例。
```yml

image: "node:12.13.0-alpine"

# 预执行的默认脚本，每个job都会最先执行这里的内容，一般用来执行通用的初始化或者预检命令
before_script:
  - echo 'do something'

# 分为3个构建阶段
stages:
  - install # 安装
  - tslint_test # 测试
  - deploy # 部署
# 定义缓存，可以在每个job复用，加速构建速度
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
# 安装步骤
prepare: 
  stage: install
  script:
  - npm install --registry https://registry.npm.taobao.org
# tslint 代码风格检查
tslint:
  stage: tslint_test
  script:
  - npm run tslint
# 执行单元测试
test:
  stage: tslint_test
  script:
  - npm run test
# 执行部署
prod_deploy:
  stage: deploy
  script:
  - echo 'deploy done'

```

# 配置 `Runner`

在GitLab中，Runners将会运行你在.gitlab-ci.yml中定义的jobs。Runner可以是虚拟机，VPS，裸机，docker容器，甚至一堆容器。

一个Runner可以服务GitLab中的某个特定的项目或者是多个项目。如果它服务所有的项目，则被称为共享的Runner。


为了有一个功能性的Runner，你需要遵循以下步骤：
```
  1.安装
  2.注册
```
### Step.1

目前采用 [Install as a Docker service](https://docs.gitlab.com/runner/install/docker.html#docker-image-installation) 进行安装。

```bash
docker run -d --name gitlab-runner --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
```

### Step.2

采用`docker exec` 调用容器命令 `gitlab-runner register` 进行注册，使用你的项目信息替换以下示例命令的部分内容。
```bash
docker exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --executor "docker" \
  --docker-image alpine:latest \
  --url "https://gitlab.com/" \
  --registration-token "PROJECT_REGISTRATION_TOKEN" \
  --description "docker-runner" \
  --tag-list "docker,aws" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"
```

进行了以上两步后，就可以在项目的 `Settings>>CI/CD` 看到 `Runner` 的信息了。
![Runner demo](https://docs.gitlab.com/ee/ci/quick_start/img/runners_activated.png)

# `Git Push`

代码推送到仓库后，可以看到`Runner`会自动开始执行`.gitlab-ci.yaml`内定义的jobs。

![pipline demo](https://docs.gitlab.com/ee/ci/quick_start/img/pipelines_status.png)