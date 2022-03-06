## 开启容器
docker run \
  -u root \
  -d \
  --name jenkins \
  --restart=always \
  -p 8010:8080 \
  -p 50000:50000 \
  -v ~/jenkins:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkinsci/blueocean

## 密码
2021年8月13日，密码验证的支持被删除。请使用个人访问令牌。

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

## 此命令之后拉一下代码，之后不需要一直输入密码
git config --global credential.helper store

## 进入容器内部
docker exec -it jenkins /bin/bash

## Jenkinsfile
```
pipeline {
    agent {
        docker {
            image 'maven:3-alpine'
            args '-v /root/.m2:/root/.m2'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
    }
}
```