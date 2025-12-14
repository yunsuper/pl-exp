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
    // 👈 수정: JNLP 연결 문제 해결을 위해 표준 Jenkins Agent 이미지로 교체
    image: jenkins/inbound-agent:latest
    args:
    - $(JENKINS_AGENT_NAME)
    - $(JENKINS_SECRET)
    - -url
    - $(JENKINS_URL)
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
                    // 1. 빌드
                    container('dind') { 
                        sh """
                        docker build -t ${dockerImageName} .
                        docker tag ${dockerImageName}:latest registry.hub.docker.com/${dockerImageName}:latest
                        """
                    }
                    
                    // 2. 로그인, 푸시, 그리고 정리 (finally 블록에 logout을 넣어 컨텍스트 유지)
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                        try {
                            container('dind') {
                                sh "docker push registry.hub.docker.com/${dockerImageName}:latest"
                            }
                        } finally {
                            // 크리덴셜 사용 직후, Pod 내부에서 로그아웃
                            container('dind') { 
                                sh 'docker logout' 
                            }
                        }
                    }
                }
            }
        }
        stage('deploy application on kubernetes cluster'){
            steps{
                container('jnlp') { 
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
        success {
            echo 'Cleanup status: Docker logout was executed in the previous stage.'
        }
    }
}
