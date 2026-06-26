pipeline {
    agent any

    stages {
        stage('SonarQube Analysis') {
            steps {
                script {
                    // Pastikan Anda menamai tool scanner Anda 'sonar-scanner' di Jenkins (Manage Jenkins > Tools)
                    def scannerHome = tool 'sonar-scanner'
                    
                    // Pastikan Anda menamai server SonarQube Anda 'SonarQubeLokal' di Jenkins (Manage Jenkins > System)
                    withSonarQubeEnv('SonarQubeLokal') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
    }
}
