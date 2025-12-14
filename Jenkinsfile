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
                    // 1. 빌드 (dind 컨테이너 사용)
                    container('dind') { 
                        sh """
                        docker build -t ${dockerImageName} .
                        docker tag ${dockerImageName}:latest registry.hub.docker.com/${dockerImageName}:latest
                        """
                    }
                    
                    // 2. 로그인 및 푸시, 그리고 정리 (logout)
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                        try {
                            container('dind') {
                                sh "docker push registry.hub.docker.com/${dockerImageName}:latest"
                            }
                        } finally {
                            // 👈 최종 해결: 크리덴셜 사용 직후, 동일한 Pod/Node 컨텍스트 내에서 로그아웃
                            // post로 분리하지 않고, withRegistry의 논리적 끝에서 정리
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
    // post 섹션을 완전히 제거합니다. (로그아웃 로직이 스테이지 내부로 이동)
    post{
        // 이전 오류를 반복하지 않기 위해 이 부분을 비우거나 제거합니다.
    }
}
