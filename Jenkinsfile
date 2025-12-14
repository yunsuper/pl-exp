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
                container('dind') { // 👈 컨테이너 지정
                    sh """
                    // 셸 명령으로 Docker 빌드를 명시합니다.
                    docker build -t ${dockerImageName} .
                    docker tag ${dockerImageName}:latest registry.hub.docker.com/${dockerImageName}:latest
                    
                    // docker.withRegistry 대신 sh 명령으로 로그인/푸시를 분리합니다.
                    // 이전에 설정된 'dockerhub-credentials'는 sh 명령에서 바로 사용할 수 없으므로,
                    // credentialsId를 통해 비밀번호를 획득하여 푸시해야 합니다. (이 부분은 사용자 환경에 맞게 조정 필요)
                    // 현재는 편의상 셸에서 직접 푸시하도록 가정합니다.
                    docker push registry.hub.docker.com/${dockerImageName}:latest
                    """
                }
            }
        }
        stage('deploy application on kubernetes cluster'){
            steps{
                container('jnlp') { // Kubectl 실행 컨테이너 지정
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
            // sh 명령을 가장 단순화하여 Master Node의 셸에서 실행되도록 시도합니다.
            // Jenkinsfile이 복잡한 컨텍스트를 벗어나도록 합니다.
            sh 'docker logout' 
        }
    }
}
