pipeline {
    
    environment {
        IMAGE = "$image"
        DEPLOYMENT = "kb-frontend"
        //SCANNER_HOME = tool 'SonarQube';
    }
    agent {
        kubernetes {
            label 'jenkins-slave'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: flutter
    image: cirrusci/flutter:stable
    command:
    - cat
    tty: true
  - name: kaniko
    image: registry.us-west-1.aliyuncs.com/acs/kaniko:v0.14.0
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_CONFIG
      value: /home/jenkins/.docker
    volumeMounts:
    - name: jenkins-docker-cfg
      mountPath: /home/jenkins/.docker
  - name: kubectl
    image: registry.us-west-1.aliyuncs.com/acs/kubectl:1.21.0
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_CONFIG
      value: /home/jenkins/.docker
    volumeMounts:
    - name: jenkins-k8-cfg
      mountPath: /home/jenkins/.kube
    - name: jenkins-docker-cfg
      mountPath: /home/jenkins/.docker
  volumes:
  - name: jenkins-docker-cfg
    secret:
      secretName: jenkins-docker-cfg   
  - name: jenkins-k8-cfg
    secret:
      secretName: jenkins-k8-cfg        
    '''
            defaultContainer 'jnlp'
        }
    }
    stages {
        
        /*stage('Dart Analyzer') {
            steps {
              publishChecks name: 'code-analyze', status: 'IN_PROGRESS', summary: 'Flutter code analyze started', title: 'Analyze code'
                container('flutter'){
                  script{
                    ANALYZE_LOGS = sh (
                      script: 'flutter analyze --no-fatal-warnings --no-fatal-infos' ,
                      returnStdout: true
                    )
                  }
                }
              publishChecks name: 'code-analyze', summary: "${ANALYZE_LOGS}", title: 'Analyze code'
            } 
        }
        */
        stage('Build flutter application') {
            steps {
              publishChecks name: 'code-build', status: 'IN_PROGRESS', summary: 'Flutter code build started', title: 'Building code'
                container('flutter'){
                    sh """
                        flutter build web --web-renderer html --release
                        echo ${GIT_BRANCH}
                        
                    """
                                          
                }
              publishChecks name: 'code-build', summary: 'Flutter code build complete', title: 'Building code'
            } 
        }
        stage ('Build and push docker image to container repository') {
          steps {
            publishChecks name: 'docker-push', status: 'IN_PROGRESS', summary: 'Building docker image and pushing to container repository', title: 'Building docker container'
              script {
                  if (GIT_BRANCH == 'origin/dev') {
                      env.STAGE = "dev"
                  } else if (GIT_BRANCH == 'origin/qa') {
                      env.STAGE = "qa"
                  } else if (GIT_BRANCH == 'origin/prod') {
                      env.STAGE = "prod"
                  } else if (GIT_BRANCH == 'origin/kube-deployment') {
                      env.STAGE = "kube-deployment"
                  }
              }
              container('kaniko') {
                  sh 'echo ${STAGE}'
                  sh 'kaniko -f ${WORKSPACE}/Dockerfile -c `pwd` --insecure --skip-tls-verify --destination=kbcontainer/kbdeployment:${STAGE}'
              }
            publishChecks name: 'docker-push', summary: 'Image built and pushed to container registery', title: 'Building docker container'
          }
        }
        stage('Deploy to Kubernetes') {
          steps {
            publishChecks name: 'kube-deploy', status: 'IN_PROGRESS', summary: "Deploying to ${STAGE} environment..."
              container('kubectl'){
                  sh("sed -i  's#IMAGE#${IMAGE}:${STAGE}#g' ${WORKSPACE}/kube-deployment.yaml")
                  sh("sed -i  's#DEPLOYMENT#${DEPLOYMENT}-${STAGE}#g' ${WORKSPACE}/kube-deployment.yaml")
                  sh("sed -i  's#GIT_COMMIT#${GIT_COMMIT}#g' ${WORKSPACE}/kube-deployment.yaml")
                  sh("cat ${WORKSPACE}/kube-deployment.yaml")
                  script {
                    try {
                      sh("kubectl apply -f ${WORKSPACE}/kube-deployment.yaml")
                    }
                    catch (Exception e) {
                      sh("kubectl create -f ${WORKSPACE}/kube-deployment.yaml")
                      echo 'Exception occurred: ' + e.toString()
                    }
                  }
              }
            publishChecks name: 'kube-deploy', summary: "Deployed to ${STAGE} environment.", title: 'Building docker container'
          }
        }
    } 
}
