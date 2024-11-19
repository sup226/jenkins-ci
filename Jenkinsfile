// 필요한 변수를 선언할 수 있다. (내가 직접 선언하는 변수, 젠킨스 환경변수를 끌고 올 수 있음)
def ecrLoginHelper="docker-credential-ecr-login" // ECR credential helper 이름
def region="ap-northeast-2"
def ecrUrl="761018868501.dkr.ecr.ap-northeast-2.amazonaws.com"
def repository="my-app-image"
def deployHost = "172.31.18.212"

// 젠킨스의 선언형 파이프라인 정의부 시작 (그루비 언어)
pipeline {
    agent any // 어느 젠킨스 서버에서도 실행이 가능

    stages {
        stage('Pull Codes from Github'){ // 스테이지 제목 (맘대로 써도 됨.)
            steps{
                checkout scm // 젠킨스와 연결된 소스 컨트롤 매니저(git 등)에서 코드를 가져오는 명령어
            }
        }
        stage('Build Codes by Gradle') {
            steps {
                sh """
                ./gradlew clean build
                ls -al ./app/build/libs
                """
            }
        }
        stage('Build Docker Image & Push to AWS ECR') {
            steps {
                // withAWS를 통해 리전과 계정의 access, secret 키를 가져옴.
                withAWS(region: "${region}", credentials: "aws-key") {
                    // AWS에 접속해서 ECR을 사용해야 하는데, 젠킨스에는 aws-cli를 설치하지 않았어요.
                    // aws-cli 없이도 접속을 할 수 있게 도와주는 라이브러리 설치.
                    // helper가 여러분들 대신 aws에 로그인을 진행. 그리고 그 인증 정보를 json으로 만들어서
                    // docker에게 세팅해 줍니다. -> docker가 ECR에 push가 가능해짐.
                    sh """
                        curl -O https://amazon-ecr-credential-helper-releases.s3.us-east-2.amazonaws.com/0.4.0/linux-amd64/${ecrLoginHelper}
                        chmod +x ${ecrLoginHelper}
                        mv ${ecrLoginHelper} /usr/local/bin/

                        echo '{"credHelpers": {"${ecrUrl}": "ecr-login"}}' > ~/.docker/config.json


                        # Docker 이미지 빌드
                        docker build -t ${repository}:${currentBuild.number} .

                        # ECR 레포지토리로 태깅
                        docker tag ${repository}:${currentBuild.number} ${ecrUrl}/${repository}:${currentBuild.number}

                        # ECR로 푸시
                        docker push ${ecrUrl}/${repository}:${currentBuild.number}
                    """
                }
            }
        }
        stage('Deploy to AWS EC2 VM') {
            steps {
                sshagent(credentials: ["jenkins-ssh-key"]) {
                    //
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${deployHost} \
                     'aws ecr get-login-password --region ${region} | docker login --username AWS --password-stdin ${ecrUrl}; \
                      docker pull ${ecrUrl}/${repository}:${currentBuild.number}; \
                      docker stop \$(docker ps -q) || true; \
                      docker run -d -p 80:8080 ${ecrUrl}/${repository}:${currentBuild.number};'
                    """
                }
            }
        }
    }
}