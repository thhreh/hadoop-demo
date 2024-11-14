pipeline {
    agent any
    
    environment {
        CLUSTER_NAME = "hadoop-cluster"
        REGION = "us-central1"
        GCS_BUCKET = "dataproc-staging-jenkins1-441202"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    
    stages {
        stage('Install Maven') {
            steps {
                script {
                    def mvnCheck = sh(script: 'which mvn', returnStatus: true)
                    if (mvnCheck != 0) {
                        echo "Installing Maven..."
                        sh '''
                            sudo apt-get update
                            sudo apt-get install -y maven
                        '''
                    } else {
                        echo "Maven is already installed"
                        sh 'mvn --version'
                    }
                }
            }
        }
        
        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }
        
        stage('Build JAR') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(installationName: 'SonarQube') { 
                    sh '''
                        mvn clean verify sonar:sonar \
                            -Dsonar.projectKey="javawebapp" \
                            -Dsonar.login="admin" \
                            -Dsonar.password="admin"
                    '''
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Upload JAR to GCS') {
            steps {
                sh '''
                    gsutil cp target/custom-wordcount-1.0-SNAPSHOT.jar gs://${GCS_BUCKET}/jars/
                '''
            }
        }
        
        stage('Submit WordCount Job to Dataproc') {
            steps {
                sh '''
                    # Submit the custom WordCount job
                    gcloud dataproc jobs submit hadoop \
                        --cluster=${CLUSTER_NAME} \
                        --region=${REGION} \
                        --class=com.example.hadoop.WordCount \
                        --jars=gs://${GCS_BUCKET}/jars/custom-wordcount-1.0-SNAPSHOT.jar \
                        -- gs://${GCS_BUCKET}/input/text_input.txt gs://${GCS_BUCKET}/output/wordcount-output
                    
                    # Wait for the job to finish
                    sleep 10
                '''
            }
        }
    }
    
    post {
        always {
            deleteDir()
        }
    }
}
