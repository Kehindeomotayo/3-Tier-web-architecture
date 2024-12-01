pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                sshagent(['credential-id']) {
                    git url: 'git@github.com:Kehindeomotayo/3-Tier-web-architecture.git', branch: 'main'
                }
            }
        }
    }
}
