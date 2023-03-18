def deploy() {
    script {
        def container = sh(
            script: "sudo docker ps -aqf 'name=flask_projet'",
            returnStdout: true
        ).trim()
        if (container) {
            sh "sudo docker stop $container && sudo docker rm $container"
            sh "sudo docker rmi  flaskapp:latest"
        }
    }
    sh 'rm -rf *'
    checkout([
        $class: 'GitSCM',
         branches: [[name: 'main']],
         userRemoteConfigs : [[
            url: 'git@github.com:Saphisto/project_traininf.git',
            credentialsId: ''
        ]]
    ])
    sh 'sudo docker build -t flaskapp -f project_dockerfile .'
    sh 'sudo docker run --name flask_projet -p 80:80 -d flaskapp:latest'
}

pipeline{
    agent {
        label 'dev'
    }
    stages{
        stage('check_if_container is running'){
            steps {
                script {
                    def container = sh(
                        script: "sudo docker ps -aqf 'name=flask_projet'",
                        returnStdout: true
                    ).trim()
                    if (container) {
                        // Stop and remove container
                        sh "sudo docker stop $container && sudo docker rm $container"
                        // Remove container image
                        sh "sudo docker rmi  flaskapp:latest"
                    }    
                }
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
        stage('if_file') {
            steps {
                script {
                    def filePath = '/home/ubuntu/workspace/check_dynamo/report.json'
                    if (!fileExists(filePath)) {
                        writeFile file: filePath, text: ''
                    }
                }    
            }    
        }    
        stage('add_username_date_test'){
            steps {
                wrap([$class: 'BuildUser']) {
                    sh 'echo "${BUILD_USER}" >> report.json'
                    script {
                        env.BUILD_USER = "${BUILD_USER}"
                        buildUser = env.BUILD_USER
                    }
                }
                script {
                    def test = sh(script: 'curl -s -o /dev/null -w "%{http_code}" localhost:80', returnStdout: true)
                        env.TEST = test
                }
                script {
                    def time = new Date().format("dd/MM/yyyy-HH:mm", TimeZone.getTimeZone('GMT+2'))
                        env.TIME = time
                }
                script {
                    sh """
                        echo "$TIME" >> report.json
                        echo "$TEST" >> report.json
                    """
                }
                withAWS(credentials:'AWS_credentials', region:'us-east-1') {
                    script {
                        sh "aws dynamodb put-item --table-name app_report --item '{\"user\": {\"S\": \"${buildUser}\"}, \"date\": {\"S\": \"${TIME}\"}, \"status\": {\"S\": \"${TEST}\"}}'"
                    }
                }
            }
        }
        stage('upload_test_to_s3'){
            steps {
                withAWS(credentials:'AWS_credentials', region:'us-east-1'){
                    s3Upload(file:'/home/ubuntu/workspace/check_dynamo/report.json', bucket:'sqlabs-devops-shay', path:'report.json')
                }
            }
        }
        stage('if_test_ok'){
            when {
                expression {
                    return env.TEST == '200';
                }
            }
            stages{
                stage('deploy on agent'){
                    steps{
                        script{
                            def agents = ["agent1", "agent2", "both"]
                            def agent = input message: 'Which agent to deploy?', parameters: [choice(choices: agents, name:'Agent')]
                                env.AGENT = agent
                        }
                    }
                }
                stage('when1') {
                    when {
                        expression { env.AGENT == 'agent1' }
                    }
                    agent {
                        label 'agent1'
                    }
                    steps {
                        deploy()
                    }
                }
                stage('when2') {
                    when {
                        expression { env.AGENT == 'agent2' }
                    }
                    agent {
                        label 'agent2'
                    }
                    steps {
                        deploy()
                    }
                }
                stage('both') {
                    when {
                        expression { env.AGENT == 'both' }
                    }
                    parallel {
                        stage('agent1') {
                            agent {
                                label 'agent1'
                            }
                            steps {
                                deploy()
                            }
                        }
                        stage('agent2') {
                            agent {
                                label 'agent2'
                            }
                            steps {
                                deploy()
                            }
                        }
                    }
                }
            }
        }
    }
}
