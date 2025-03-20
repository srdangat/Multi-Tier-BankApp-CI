pipeline {
    agent any

    tools {
        maven 'maven3' // Define the Maven tool version to use
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner' // Define the SonarQube scanner tool path
        IMAGE_TAG = "v${BUILD_NUMBER}" // Set the image tag to the build number (v1, v2, etc.)
    }

    stages {
        stage('Git Checkout') {
            steps {
                // Checkout the latest code from the 'main' branch
                git branch: 'main', url: 'https://github.com/srdangat/Multi-Tier-BankApp-CI.git'
            }
        }

        stage('Compile') {
            steps {
                // Compile the code using Maven
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                // Run unit tests using Maven
                sh 'mvn test'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                // Scan the project file system for vulnerabilities using Trivy
                sh 'trivy fs --format table -o Fs-report.html .'
            }
        }

        stage('SonarQube Scan') {
            steps {
                // Run the SonarQube scanner to analyze the code quality
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=GCBank \
                        -Dsonar.projectKey=GCBank \
                        -Dsonar.java.binaries=target''' // Run SonarQube scan
                }
            }
        }

        stage('QualityGateCheck') {
            steps {
                // Wait for the SonarQube quality gate check (timeout set to 1 hour)
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') {
            steps {
                // Package the application using Maven
                sh 'mvn package'
            }
        }

        stage('Publish Artifact') {
            steps {
                // Publish the built artifact to the repository
                withMaven(globalMavenSettingsConfig: 'sanket', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image with the specified tag
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t srdangat/gcbank:$IMAGE_TAG ."
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                // Scan the built Docker image for vulnerabilities using Trivy
                sh "trivy image --format table -o image-report.html srdangat/gcbank:$IMAGE_TAG"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    // Push the Docker image to the registry
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push srdangat/gcbank:$IMAGE_TAG"
                    }
                }
            }
        }

        stage('Update Manifest File in Multi-Tier-BankApp-CD') {
            steps {
                script {
                    // Clean workspace before starting
                    cleanWs()

                    // Clone the repository using GitHub credentials
                    git branch: 'main', credentialsId: 'github-token', url: 'git@github.com:srdangat/Multi-Tier-BankApp-CD.git'

                    // Update the manifest file with the new image tag
                    sh ''' 
                        sed -i "s|srdangat/gcbank:.*|srdangat/gcbank:${IMAGE_TAG}|" bankapp/bankapp-ds.yml
                        echo "Updated the image tag in bankapp-ds.yml"
                        cat bankapp/bankapp-ds.yml
                    '''

                    // Commit and push changes to the repository
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"
                        git add bankapp/bankapp-ds.yml
                        git commit -m "Updated the image tag to ${IMAGE_TAG}"
                        git push origin main
                    '''
                }
            }
        }
    }
}
