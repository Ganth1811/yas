pipeline {
    agent any
    tools { 
        maven 'Maven_3_9'
        jdk 'JDK_21' 
        snyk 'Snyk' // Gọi tool Snyk đã cấu hình
    }

    stages {
        // --- YÊU CẦU NÂNG CAO C: Quét bảo mật toàn cục ---
        stage('Security Scan') {
            parallel { // Chạy song song để tiết kiệm thời gian
                stage('Gitleaks (Secrets)') {
                    steps { 
                        sh 'docker run -v $(pwd):/path zricethezav/gitleaks:latest detect --source="/path" -v || true' 
                    }
                }
                stage('Snyk (Vulnerabilities)') {
                    steps {
                        // Gọi Snyk quét toàn bộ các file pom.xml trong monorepo
                        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                            sh 'snyk test --all-projects || true' 
                        }
                    }
                }
            }
        }

        // --- YÊU CẦU 6: CI cho Media Service ---
        stage('CI: Media Service') {
            when { changeset "media/**" }
            stages {
                stage('Test & Coverage') {
                    steps { dir('media') { sh 'mvn clean test jacoco:report' } }
                    post {
                        always { junit 'media/target/surefire-reports/*.xml' } 
                        success {
                            // Yêu cầu 7b: Chặn pipeline nếu Test Coverage < 70%
                            jacoco execPattern: 'media/target/jacoco.exec', changeBuildStatus: true, minimumInstructionCoverage: '70'
                        }
                    }
                }
                stage('SonarQube') {
                    steps { 
                        dir('media') { 
                            withSonarQubeEnv('SonarQube_Server') { sh 'mvn sonar:sonar' } 
                        } 
                    }
                }
                stage('Build') {
                    steps { dir('media') { sh 'mvn clean package -DskipTests' } } 
                }
            }
        }
        
        // --- (Lặp lại cấu trúc trên cho Product, Cart, Order...) ---
    }
}