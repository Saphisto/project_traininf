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
        stage('check if container is already running'){
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
        stage('build dockerfile'){
            steps {
                sh 'sudo docker build -t flaskapp -f project_dockerfile .'
            }
        }
        stage('run flask container'){
            steps {
                sh 'sudo docker run --name flask_projet -p 80:80 -d flaskapp:latest'
            }
        }
        stage('check if report file exist') {
            steps {
                script {
                    def filePath = '/home/ubuntu/workspace/check_dynamo/report.json'
                    if (!fileExists(filePath)) {
                        writeFile file: filePath, text: ''
                    }
                }    
            }    
        }    
        stage('add usrname, date and test result to report file'){
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
        stage('upload report file to S3'){
            steps {
                withAWS(credentials:'AWS_credentials', region:'us-east-1'){
                    s3Upload(file:'/home/ubuntu/workspace/check_dynamo/report.json', bucket:'sqlabs-devops-shay', path:'report.json')
                }
            }
        }
        stage('check if the test is successful'){
            when {
                expression {
                    return env.TEST == '200';
                }
            }
            stages{
                stage('choose which agents to deploy if test successful'){
                    steps{
                        script{
                            def agents = ["agent1", "agent2", "both"]
                            def agent = input message: 'Which agent to deploy?', parameters: [choice(choices: agents, name:'Agent')]
                                env.AGENT = agent
                        }
                    }
                }
                stage('deploy on agent1') {
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
                stage('deploy on agent2') {
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
                stage('deploy on both agents') {
                    when {
                        expression { env.AGENT == 'both' }
                    }
                    parallel {
                        stage('deploy on agent1') {
                            agent {
                                label 'agent1'
                            }
                            steps {
                                deploy()
                            }
                        }
                        stage('deploy on agent2') {
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
