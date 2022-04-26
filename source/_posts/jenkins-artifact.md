---
title: jenkins archiveArtifacts的使用
date: 2021-10-12 15:23:13
tags:
- jenkins
- devops
---

有时我们需要在多个stage 中传递制品使用，尤其是使用 master - slave 架构，agent docker 方式，当多个机器打了相同的 tag， 调度时会调到不同的节点上，上个stage 可能在 A 机器，下面stage 可能到了B机器，在B中就找不到A 中生成的物。



我们使用 archiveArtifacts，来解决上述问题。它会在 jenkins-master 上 生成 artifact，在 stage 中执行 `copyArtifacts` 会将 生成的 artifact 拉过去.


<!-- more -->


# 1.所需插件

- Copy Artifact
- ~~[job cacher](https://plugins.jenkins.io/jobcacher/) 缓存测试没成功~~

[插件离线地址](http://updates.jenkins-ci.org/download/plugins)

​      

# 2.jenkinsfile 示例


```json
pipeline {
    agent none
    
    stages {
        stage('archivePush') {
            agent {
                docker {
                    image 'maven:3-alpine'  
                }
            }
 
            steps {
                // 生成测试文件
                sh "mkdir -p mydata; echo 'hello world ....' >> mydata/test.txt "
                // 压缩打包
                zip zipFile: 'test.zip', archive: false, dir: 'mydata'
            }
            post {
                success {
                    // artifact
                    archiveArtifacts artifacts: 'test.zip', fingerprint: true
                }
            }
        }

        stage('archivePull') {
            agent {
                docker {
                    image 'maven:3-alpine' 
                }
            }
 
            steps {
                // 拉取 artifact
                copyArtifacts filter: 'test.zip', fingerprintArtifacts: true, projectName: '${JOB_NAME}', selector: specific('${BUILD_NUMBER}') 
                // 解压
                unzip zipFile: 'test.zip', dir: './archive_new' 
                // 查看
                sh "cat archive_new/test.txt"
            }
           
        }
    }
}

```





# 3.参考:

- [Jenkins Pipeline——暂存文件，以用于之后的构建](https://blog.csdn.net/u013670453/article/details/115711871)
- [copyArtifact](https://www.jenkins.io/doc/pipeline/steps/copyartifact/)

