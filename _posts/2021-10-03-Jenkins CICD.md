---
title: "CI/CD with Jenkins"    
layout: single    
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: Jenkinsfile을 이용한 파이프라인 구축
last_modified_at: 2021-10-03     
---



## CI란?

CI는 Continuous Integration의 약자로 지속적인 통합을 의미하는데 이는 빌드/테스트 과정을 말합니다.

여러 개발자가 작업할 때 발생하는 변경사항에 대해 자동으로 빌드를 진행하고 테스트하여 공유 리포지토리에 통합되므로 충돌과 같은 문제를 해결할 수 있습니다.

CI를 사용함으로써 유지보수와 코드 품질 개선에 긍적적 효과를 가져올 수 있습니다.



## CD란?

CD는 Continuous Deployment의 약자로 지속적 배포를 의미하며 프로젝트의 변경사항을 리토지토리에서 운영서버까지 자동으로 릴리즈하는 과정을 말합니다.

CD를 사용하면 짧은 주기로 프로젝트의 배포가 이루어질 때 배포시간을 단축해주어 생산성 향상에 도움을 줍니다.



프로젝트의 변경사항에 대해 빌드와 테스트를 진행하고 이를 통과하면 배포를 진행하는과정을 CI/CD라고 부릅니다. 최근 애자일 방법론을 사용하여 많은 개발이 이루어지고 있다보니 CI/CD의 필요성이 매우 높아져 있습니다.



## CI/CD Tools

CD/CD 서비스를 제공하는 대표적인 도구로 Jenkins, Travis CI, CircleCI 등이 존재합니다.



![1](/assets/image/cicd_jenkins/1.png)

Jenkins는 Java 기반의 지속적 통합 서비스를 제공하는 툴입니다. 많은 사용자를 보유하고있어 관련 문서가 많이 존재하고 다양한 플러그인이 있습니다.

무료로 사용할 수 있지만 직접 호스팅을 해야하기때문에 추가적인 인프라 비용이 발생합니다.



![2](/assets/image/cicd_jenkins/2.png)

Travis CI는 Travis에서 만든 CI툴로 .travis.yml라는 어떠한 작업을 수행할지 정의한 설정 파일을 프로젝트 디렉토리에 저장한 후 github과 연동시키면 CI/CD과정을 자동으로 수행하고 결과를 알려줍니다. Jenkins에 비해 사용하기위한 설정과정이 단순하고 추가적인 인프라를 구성할 필요가 없다는 장점이 있습니다.



![3](/assets/image/cicd_jenkins/3.png)

CircleCI는 클라우드 기반 시스템으로 추가적인 인프라 구성이 필요하지 않고 Travis CI처럼 단순한 설정을 통해 사용할 수 있습니다. 일정 한도 내에서만 무료로 사용이 가능하다는 점이 있습니다.



이 중  Jenkins를 사용해 CI/CD 환경을 구성해보겠습니다.



## Jenkins를 사용한 CI/CD 파이프 라인 구축

Jenkins 서버로 GCP를 사용했습니다.

Jenkins 설치방법은 따로 다루지 않고 Jenkinsfile을 사용해 빌드 테스트 배포과정을 자동화 하는과정을 다루었습니다.



첫 번째로 Jenkinsfile을 작성해야합니다.

jenkins pipeline은 2가지 문법을 지원하는데 Declarative와 Scripted가 있습니다. Scripted문법은 자유도가 높지만 코드가 복잡하고 Declarative는 자유도가 낮지만 사용하기 쉽다는 차이가 있습니다. 복잡한 처리가 필요하지 않기 때문에 두 가지 문법중 Declarative를 사용해 Jenkinsfile을 작성했습니다.

```
pipeline {
    agent any

    tools {
        maven 'Maven 3.6.3'
    }

    stages {
        stage('poll') {
            steps {
                checkout scm // get code from remove repository
            }
        }
        stage('build') {
            steps {
                script {
                    sh 'mvn clean package -DskipTests=true';
                }
            }
        }
        stage('unit test') {
            steps {
                script {
                    sh 'mvn surefire:test';
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }
        stage('deploy') {
            steps([$class: 'BapSshPromotionPublisherPlugin']) {
                sshPublisher(
                    continueOnError: false, failOnError: true,
                    publishers: [
                        sshPublisherDesc(
                            configName: "web-instance-1",
                            verbose: true,
                            transfers: [
                                sshTransfer(
                                    sourceFiles: "target/*.jar",
                                    removePrefix: "target",
                                    remoteDirectory: "/",
                                    execCommand: "sudo fuser -k 8080/tcp; "
                                               + "nohup sudo java -jar -Dspring.profiles.active=prod > nohup.out 2>&1 &"
                                )
                            ]
                        )
                    ]
                )
            }
        }
    }
    post {
        success {
            slackSend (channel: '#develop', color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
        }
        failure {
            slackSend (channel: '#develop', color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
        }
    }
}
```

Jenkinsfile작성 후 jenkins 서버로 접속해 해당 프로젝트의 item을 생성합니다.

![4](/assets/image/cicd_jenkins/4.png)

Jenkins를 다운로드받아 서버에 배포하면 다음과같은 화면으로 접속할 수 있습니다. 좌측 메뉴의 새로운 Item을 클릭해 작업을 만들어보겠습니다.

![5](/assets/image/cicd_jenkins/5.png)

개인적으로 개발중인 프로젝트의 CI/CD 파이프라인을 구축할 것이기 때문에 item name은 {프로젝트명}_pipeline으로 하고 타입은 Pipeline을 선택했습니다.

OK버튼을 눌러 새 아이템을 생성하면 바로 해당 아이템의 설정을 위한 화면이 뜨게됩니다.

![6](/assets/image/cicd_jenkins/6.png)

github 프로젝트가 push될때마다 빌드->테스트->배포가 자동으로 진행되도록 할것이기 때문에 Buil Triggers를 GitHub hook trigger for GITScm polling으로 선택하고 github에서 프로젝트를 불러와 프로젝트 디렉토리에 작성해놓은 Jenkinsfile을 읽어 작업을 수행할 것이기 때문에 Definition은 Pipeline script from SCM으로 하고 SCM도 Git으로 선택했습니다.

Repository URL과 Credentials은 해당 프로젝트의 github 레포지토리 주소와 해당 프로젝트에 접근하기위한 인증정보입니다.

Script Path는 프로젝트 디렉토리에 생성한 Jenkinsfile명 입니다.

작성이 완료되었으면 저장 버튼을 클릭한 후 프로젝트를 github에 push하여 테스트를 진행하면 됩니다.

![7](/assets/image/cicd_jenkins/7.png)

프로젝트 push 후 jenkins를 확인해보면 다음과 같이 빌드/테스트/배포가 수행된걸 확인할 수 있습니다.

![8](/assets/image/cicd_jenkins/8.png)

저는 성공 실패 여부에 따라 slack으로 알림을 받도록 Jenkinsfile을 작성했기 때문에 위와같은 알림을 받았습니다.

