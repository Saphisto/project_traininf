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
        stage('add_username_date_test'){
            steps {
                script {
                    def test = sh(script: 'curl -s -o /dev/null -w "%{http_code}" localhost:80', returnStdout: true)
                    env.TEST = test
                }
                script {
                    def now = new Date().format("dd/MM/yyyy-HH:mm", TimeZone.getTimeZone('GMT+2'))
                        env.NOW = now
                }
                wrap([$class: 'BuildUser']) {
                sh 'echo "${BUILD_USER}", "$NOW", "$TEST" >> sometext.csv'
                }
            }
        }
        // stage('upload_test_to_s3'){
        //     steps {
        //         withAWS(credentials:'AWS_credentials', region:'us-east-1'){
        //             s3Upload(file:'sometext.csv', bucket:'sqlabs-devops-shay', path:'sometext.csv')
        //         }
        //     }
        // }
        stage('if_test_ok'){
            when {
                expression {
                    return env.TEST == '200';
                }
            }
            stages{
                stage('checkout_scm_on_agent'){
                    steps{
                        script{
                            def agents = ["agent1", "agent2", "both"]
                            def agent = input message: 'Which agent to deploy?', parameters: [choice(choices: agents, name:'Agent')]
                            node(agent){
                                stage('Checkout_SCM_on_selected_agent'){
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
                                stage('Check and Stop Container_on_selected_agent') {
                                    
                                        // Check if container is running
                                        script {
                                            def container = sh(
                                                script: "sudo docker ps -aqf 'name=flask_projet'",
                                                returnStdout: true
                                            ).trim()
                                            if (container) {
                                                // Stop and remove container
                                                sh "sudo docker stop $container && sudo docker rm $container"
                                                // Remove container image
                                                sh "sudo docker rmi flaskapp:latest"
                                            }
                                        }
                                    
                                }                                                
                                stage('deploy_on_selected_agent'){
                                    sh 'sudo docker build -t flaskapp -f project_dockerfile .'
                                }
                                stage('run_on_selected_agent'){
                                    sh 'sudo docker run --name flask_projet -p 80:80 -d flaskapp:latest'
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
