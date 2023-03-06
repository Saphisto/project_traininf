pipeline{
    agent {
        label 'agent1'
    }
    stages{
        stage('STOP'){
            steps {
                sh """
                sudo docker stop flask_projet
                sudo docker rm flask_projet
                sudo docker rmi flaskapp:latest
                """
            }
        }
        stage('Checkout SCM'){
            steps {
                sh 'rm -rf *'
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs : [[
                        url: 'git@github.com:Saphisto/project_traininf.git',
                        credentialsId: ''
                    ]]
                ])
            }
            
        }
        stage('builddockerfile'){
            steps {
                sh 'sudo docker build -t flaskapp -f project_dockerfile .'
            }
        }
        stage('run_container'){
            steps {
                sh 'sudo docker run --name flask_projet -p 80:80 -d flaskapp:latest'
            }
        }
        stage('add_username_date_test'){
            steps {
                script {
                    def now = new Date().format("dd/MM/yyyy-HH:mm", TimeZone.getTimeZone('GMT+2'))
                        env.NOW = now
                }
                wrap([$class: 'BuildUser']) {
                sh 'echo "${BUILD_USER}", "$NOW" >> sometext.csv'
                }
                sh """
                curl -LI localhost:80 -o /dev/null -w '%{http_code}' -s >> sometext.csv
                """
            }
        }
        stage('upload_test_to_s3'){
            steps {
                withAWS(credentials:'AWS_credentials', region:'us-east-1'){
                    s3Upload(file:'sometext.csv', bucket:'sqlabs-devops-shay', path:'sometext.csv')
                }
            }
        }
    }
}
