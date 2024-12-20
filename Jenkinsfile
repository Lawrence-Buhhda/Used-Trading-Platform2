pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        GITHUB_URL = "https://github.com"
        GITHUB_ORG = "Lawrence-Buhhda"
        GITHUB_REPO = "Used-Trading-Platform2"
    }
    
    stages {
        stage('Set Git Proxy') {
            steps {
                script {
                    // 设置 Git 代理
                    sh 'git config --global http.proxy "http://192.168.71.59:10809"'
                    sh 'git config --global https.proxy "https://192.168.71.59:10809"'
                }
            }
        }
        
        stage('Configure Git') {
            steps {
                sh 'git config --global http.version HTTP/1.1'
                sh 'git config --global http.postBuffer 1048576000'
                sh 'git config --global https.postBuffer 1048576000'
            }
        }
        
        stage('Checking if there is a running build') {
            steps {
                script {
                    def buildNumber = env.BUILD_NUMBER as int
                    if (buildNumber > 1) {
                        // 如果不是第一次构建，等待上一次构建的里程碑
                        milestone(buildNumber - 1)
                    }
                    // 当前构建完成后，创建当前构建的里程碑
                    milestone(buildNumber)
                    // 构建步骤...
                }
            }
        }
        
        stage('Run Tests') {
            when {
                changeRequest()
            }
            steps {
                script {
                    echo 'Running tests for the change request.'
                }
            }
        }
        
        stage('Git Checkout') {
            when {
                not { changeRequest() }
            }
            steps {
                // git branch: 'master', changelog: false, credentialsId: '705f6177-34ad-4a85-9aaf-e61f089a56a2', poll: false, url: 'https://github.com/Lawrence-Buhhda/Used-Trading-Platform2.git'   
                script {
                    def scmVars = checkout(
                                        [$class: 'GitSCM', branches: [[name: "${ghprbActualCommit}"]], 
                                        doGenerateSubmoduleConfigurations: false,
                                        submoduleCfg: [], 
                                        extensions: [
                                            [$class: 'RelativeTargetDirectory', relativeTargetDir: 'codes'],
                                            [$class: 'CleanBeforeCheckout']
                                        ],
                                        userRemoteConfigs: [
                                                [
                                                    credentialsId: '705f6177-34ad-4a85-9aaf-e61f089a56a2', 
                                                    name: 'origin', 
                                                    refspec: '+refs/pull/*:refs/remotes/origin/pr/*', 
                                                    url: "${GITHUB_URL}/${GITHUB_ORG}/${GITHUB_REPO}.git"
                                                ]
                                            ]
                                        ]
                                    )
                    env.GIT_BRANCH = "${scmVars.GIT_BRANCH}"
                    env.GIT_COMMIT = "${scmVars.GIT_COMMIT}"
                }
            }
        }
        
        stage('COMPILE') {
            steps {
                sh "mvn clean compile"
            }
        }
        
        stage('OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'My DP Check'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }        
        
        stage('Sonarqube') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Trading-Platform \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=Trading-Platform'''
                }
            }
        }        
        
        stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
        
        stage('Package') {
            steps {
                sh "mvn package"
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: '**/target/*.war', onlyIfSuccessful: true
            }
        }
    }
    
    post {
        failure {
            mail to: '2493451720@qq.com', subject: "Jenkins Build Failed", body: "Please check the build at ${env.BUILD_URL}"
            setBuildStatus("Build failed", "FAILURE");
        }
        success {
            setBuildStatus("Build succeeded", "SUCCESS");
        }
        always {
            setBuildStatus("Build in progress", "PENDING")
        }
    }
}
// 在这里定义 setBuildStatus 方法
def setBuildStatus(String statusMessage, String status) {
    step([
        $class: 'GitHubCommitStatusSetter',
        reposSource: [$class: 'ManuallyEnteredRepositorySource', url: 'https://github.com/Lawrence-Buhhda/Used-Trading-Platform2'],
        contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: "ci/jenkins/build-status"],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: statusMessage, state: status]] ]
    ])
}
