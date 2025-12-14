pipeline {
    environment{
        dockerImageName= "yunsuper/simple-echo"
    }
    agent {
        kubernetes{
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: sheayun/jnlp-agent-sample
    env:
    - name: DOCKER_HOST
      value: "tcp://localhost:2375"
  - name: dind
    image: docker:latest
    command:
    - /usr/local/bin/dockerd-entrypoint.sh
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
'''
        }
    }
    stages{
        stage('git scm update'){
            steps{ 
                checkout scm 
            }
        }
        stage('docker build && push'){
            steps{
                script{
                    dockerImage = docker.build dockerImageName
                    // 컨테이너 컨텍스트를 명시적으로 지정하여 docker 명령을 실행합니다.
                    container('dind') { 
                        dockerImage = docker.build dockerImageName
                        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                            dockerImage.push("latest")
                        }
                    }
                }
            }
        }
        stage('deploy application on kubernetes cluster'){
            steps{
                withKubeConfig([credentialsId: 'KUBECONFIG',
                serverUrl: 'https://kubernetes.default',
                namespace: 'default']) {
                    sh '''
                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml
                    '''
                }
            }
        }
    }
    post{
        always{
            script {
                node('') { // 👈 수정: container 스텝을 node 스텝으로 감싸 Node 컨텍스트를 제공합니다.
                    container('jnlp') {
                        sh 'docker logout'
                    }
                }
            }
        }
    }
}
