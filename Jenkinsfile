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
                    container('dind') { // Docker Build는 DinD 컨테이너에서 실행
                        def dockerImage = docker.build dockerImageName
                        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                            dockerImage.push("latest")
                        }
                    }
                }
            }
        }
        stage('deploy application on kubernetes cluster'){
            steps{
                container('jnlp') { // Kubectl은 JNLP 컨테이너에서 실행 (kubectl이 설치되어 있다고 가정)
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
    }
    post{
        always{
            // 전역 post는 반드시 node로 감싸서 컨텍스트를 복구해야 합니다.
            script {
                node('') { 
                    sh 'docker logout' // docker logout은 jnlp 컨테이너에서 실행 가능
                }
            }
        }
    }
}
