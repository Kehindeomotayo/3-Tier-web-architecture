pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                sshagent(['23b40ddb-4777-4221-9737-c68e40e14f0b']) {
                    git url: 'git@github.com:Kehindeomotayo/3-Tier-web-architecture.git', branch: 'main'
                }
            }
        }
    }
}
