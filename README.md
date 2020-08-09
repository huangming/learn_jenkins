# Learn_jenkins
本项目为Jenkins+Docker+Gitlab初步学习devops运维知识实践
vagrant参考了[此项目](https://github.com/ernesen/infra-ansible)

my-nginx-alpine-v1.tar: 自建nginx镜像dogrich/my-nginx:v1

### 关键知识点
* 构建触发器
    轮询SCM就是查看源码管理的代码有没有更新，如果更新了就去构建，没有更新就不会构建。
    webhook可以在SCM有更新的时候自动触发构建

* Jenkinsfile、Pipelinne
    [用 Jenkins 构建 CI/CD 流水线](https://linux.cn/article-11546-1.html)

### 推荐入门书籍
* [Jenkins 2.x 实践指南](https://book.douban.com/subject/33407581/)


### 实践效果
```
1. 工作机(宿主机)git push 提交源代码（nginx简单index.html）变更到gitlab
2. jenkins自动在docker专用服务器git pull源代码
3. jenkins使用docker build（通过Dockerfile）模拟构建应用并打包到docker
```

### 实践环境
宿主机 `fedora30`
三台虚拟机 `ubuntu 14.04 trusty64`
```
Jenkins服务器  jenkins.sample.com
Docker服务器   docker.sample.com
Gitlab服务器   gitlab.sample.com 
```

把虚拟机假域名加进宿主机hosts
```
# add addresses to /etc/hosts 
echo "192.168.99.154 gitlab.sample.com" | sudo tee -a /etc/hosts 
echo "192.168.99.153 jenkins.sample.com" | sudo tee -a /etc/hosts 
echo "192.168.99.152 docker.sample.com" | sudo tee -a /etc/hosts 
```

### 安装gitlab
下载dep包并上传到服务器安装
```
wget -O https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/apt/packages.gitlab.com/gitlab/gitlab-ce/ubuntu/pool/trusty/main/g/gitlab-ce/gitlab-ce_11.9.12-ce.0_amd64.deb
scp gitlab-ce_11.9.12-ce.0_amd64.deb vagrant@gitlab.sample.com:~/
ssh vagrant@gitlab.sample.com
sudo dpkg -i gitlab-ce_11.9.12-ce.0_amd64.deb
sudo gitlab-ctl reconfigure
```

浏览器访问 gitlab.sample.com,会提示修改root密码，改完后以root用户登录

### 安装Jenkins
先安装java环境
然后[下载dep包](https://mirrors.tuna.tsinghua.edu.cn/jenkins/debian-stable/jenkins_2.235.3_all.deb)上传安装

启动：
```
sudo /etc/init.d/jenkins [start|restart|stop]
```

浏览器打开网页 `jenkins@sample.com:8080` 后
一直显示 `Please wait while Jenkins is getting ready to work ...`
`dpkg -L jenkins` 查看安装路径
```
修改 /var/lib/jenkins/hudson.model.UpdateCenter.xml
https://updates.jenkins.io/update-center.json
更改为https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
是国内的清华大学的镜像地址。
或者更改为http://updates.jenkins.io/update-center.json，即去掉 https 中的 s 。

sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' /var/lib/jenkins/updates/default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' /var/lib/jenkins/updates/default.json

然后重启服务
```

修改jenkins端口
```
$ sudo vim /etc/default/jenkins
#修改如下内容
HTTP_PORT=8085
#重启jenkins服务
$ sudo /etc/init.d/jenkins restart
```

### 实践devops项目
#### gitlab上新建nginx-test项目
宿主机：
```
cat ~/.ssh/id_rsa.pub 
```
把公钥拷贝，到gitlab个人配置sshkeys新增，然后clone项目修改并提交
如果宿主机还没有公钥，执行`ssh-keygen -t rsa`生成
```
git clone git@gitlab.sample.com:hming/nginx-test.git
echo "hello world" >> index.html
git add .
git commit -m "first commit"
git push
```

#### jenkins部署项目
安装jenkins插件
* Publish Over SSH

在jenkins服务器，切换到jenkins用户，生成公钥并发送到gitlab
```
sudo su -s /bin/bash jenkins
ssh-keygen -t rsa
ssh-copy-id -i vagrant@gitlab.sample.com 
```

`登录Jenkins` => `系统管理` => `系统配置` => `Publish over SSH`
SSH Servers
```
Name:docker.sample.com
HostName:docker.sample.com
Remote Directory:/home/vagrant
高级-Password:设置为vagrannt
```

1. 源代码管理
选git，Repository Url:
```
git@gitlab.sample.com:hming/nginx-test.git
```

2. 构建触发器
选 Build when a change is pushed to GitLab. 
webhook URL:http://jenkins.sample.com:8080/project/nginx-test

3. 构建环境
选 Send files or execute commands over ssh
```
ssh server: docker.sample.com
Transfers: 
    Remote directory: /
    Exec command:
        cd /home/vagrant/project/nginx-test
        git pull
```
 
4. 构建
选 Send files or execute commands over ssh
```
ssh server: docker.sample.com
Transfers: 
    Remote directory: /
        cd /home/vagrant/project/nginx-test
        sudo docker images -f "dangling=true" -q | uniq | xargs -I {} sudo docker rmi --force {}
        sudo docker ps | grep "mynginx-test:last" | awk '{print  $1}'|sed 's/%//g' | uniq | xargs -I {} sudo docker stop {}
        sudo docker ps -a | grep "mynginx-test:last" |awk '{print  $1}'|sed 's/%//g' | uniq | xargs -I {} sudo docker rm {}
        docker build -t mynginx-test:last .
        docker run -p 8080:80 -d mynginx-test:last 
```

### 问题
1. gitlab部署完成后打开网页修改密码报错 `GitLab is taking too much time to respond`
    gitlab服务器要4G以上内存！否则可能引起上述错误

2. `copying bootstrap data to pipe caused \"write init-p: broken pipe\"": unknow`
    在宿主机build和run没问题，在docker机器上报这个错。docker版本与内核不兼容问题,重新安装docker
    ```
    sudo apt-get install docker-ce=18.03.0~ce-0~ubuntu
    https://github.com/docker/for-linux/issues/595
    ```

3. gitlab上设置webhook的时候报错 `Url is blocked: Requests to the local network are not allowed`
    root用户登录 `Admin area => Settings => Netwwork => Outbound requests` 勾上 `Allow requests to the local network from hooks and services`

4. gitlab上配置webhook后测试报错 `Hook executed successfully but returned HTTP 403` 
    进入jenkins：
    * `Manage Jenkins` -> `Configure Global Security` -> `授权策略` -> `Logged-in users can do anything （登录用户可以做任何事情） 点选` -> `匿名用户具有可读权限 点选`
    * 去掉跨站点请求伪造 点选 放开
    `Manage Jenkins` -> `Configure Global Security` -> `CSRF Protection（跨站请求伪造保护）`
    * 去掉`Gitlab enable authentication` 点选 放开
    `系统管理` -> `系统设置` -> `Enable authentication for '/project' end-point`
    
### 流水线 Pipeline    
安装插件:
* `SSH Pipeline Steps`
* `ConditionalBuildStep`
 
远程ssh(注意$、'等转义):
```
pipeline { 
    agent any 
    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        disableConcurrentBuilds()
        timeout(time:5, unit: 'MINUTES')
    }
    parameters {
        choice(name: 'CHOICES', choices: 'dev\ntest\nproduct', description: '请选择部署的环境')
    }
    triggers {
        // cron('30 16 * * *') // 定时任务
        // pollSCM('H/1 * * * *') // 每分钟判断代码是否变化
        gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All',
        secretToken: "ec4a11ba90ef6cc3ae28a1ace6d28a00")
    }
    stages {
        stage('checkout') { 
            steps { 
                script {
                    def url = 'git@gitlab.sample.com:hming/nginx-test.git'
                    def remote = [:]
                    remote.name = 'docker'
                    remote.host ='docker.sample.com'
                    remote.allowAnyHosts= true
  	                withCredentials([usernamePassword(credentialsId: 'docker_ssh', passwordVariable: 'password', usernameVariable: 'username')]) {
   	                	remote.user = "${username}"
   	                	remote.password = "${password}"
  	                }
                    def checkoutcommandStr="""
                      if [ ! -d "/home/vagrant/project/nginx-test" ]; then
                        cd /home/vagrant/project
                        git clone ${url}
                      fi
                      cd /home/vagrant/project/nginx-test
                      git pull
                    """
                    sshCommand remote: remote, command: checkoutcommandStr
                }
            }
        }
        stage('Build') { 
            steps { 
                script {
                    def remote = [:]
                    remote.name = 'docker'
                    remote.host ='docker.sample.com'
                    remote.allowAnyHosts= true
  	                withCredentials([usernamePassword(credentialsId: 'docker_ssh', passwordVariable: 'password', usernameVariable: 'username')]) {
   	                	remote.user = "${username}"
   	                	remote.password = "${password}"
  	                }
                    def commandStr = """
                      cd /home/vagrant/project/nginx-test
                      docker images -f "dangling=true" -q | uniq | xargs -I {} docker rmi --force {}
                      docker ps | grep "mynginx-test:last" | awk '{print \$1}'|sed 's/%//g' | uniq | xargs -I {} docker stop {}
                      docker ps -a | grep "mynginx-test:last" |awk '{print \$1}'|sed 's/%//g' | uniq | xargs -I {} docker rm {}
                      docker build -t mynginx-test:last .
                      docker run -p 8080:80 -d mynginx-test:last
                    """
                    sshCommand remote: remote, command: commandStr
                }
            }
            post {
                always {
                    echo "stage post always"
                }
            }
        }
        stage('Test'){
            steps {
                sh 'echo Test'
            }
        }
        stage('PreTest'){
            steps {
                script {
                    approvalMap = input(
                        message: '准备发布到哪个环境？',
                        ok: '确定',
                        parameters: [
                            choice(choices: 'dev\ntest\nproduct', description: '发布到什么环境?', name: 'ENV'),
                            string(defaultValue: '', description: '', name: 'myparam')
                        ],
                        submitter: 'admin,hming,root',
                        submitterParameter: 'APPROVER'
                    )
                }
            }
        }
        stage('Deploy to test') {
            when {
                expression { return params.CHOICES == 'test' }
            }
            steps {
                echo "Deploy to test"
            }
        }
        stage('Deploy to prod') {
            steps {
                echo "Deploy to prod"
                echo "操作者是 ${approvalMap['APPROVER']}"
                echo "发布到什么环境？ ${approvalMap['ENV']}"
                echo "自定义参数: ${approvalMap['myparam']}"
            }
        }
    }
    post {
        changed {
            echo "pipelinne post changed"
        }
        always {
            echo "pipelinne post always"
        }
        success {
            echo "pipelinne post success"
        }
    }
}
```



