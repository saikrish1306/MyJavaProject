@Library('startup-library') _

pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME= tool 'sonar-scanner'
        CLOUDSDK_CORE_PROJECT = 'project-1-449106'
    }

    stages {
       // stage('Git Checkout') {
       //     steps {
       //        git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/rajalearn90/DevopsJavaProject.git'     
       //     }
       // }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

      //install trivy in the jenkins server
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

         // add the webhook in the sonarqube server http://34.47.215.174:8080/sonarqube-webhook
         //Create token in the Sonarqube server
         //add server in the manage jenkins
         //add environement for SONAR_HOME

        stage('SonarQube Analsyis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame -Dsonar.java.binaries=. '''
                }
            }
        }
                 
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred' 
                }
            }
        }
        
        stage('Build') {
            steps {
               sh "mvn package"
            }
        }
        
        stage('Publish To Nexus') {
            steps {
               withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
               }
            }
        }
        
        stage ('GCR Auth'){
            steps{
                withCredentials([file(credentialsId: 'gcrsvc-cred', variable: 'gcrsvc_cred')]) {
                    sh '''
                    gcloud auth activate-service-account --key-file="$gcrsvc_cred"
                    gcloud config set project project-1-449106
                    '''
                }
            }
        }
        
        stage('Build and create docker image'){
            steps{
                sh 'docker build -t us-central1-docker.pkg.dev/project-1-449106/jenkins-project/boardgame:${BUILD_NUMBER} . '
            }
        }
        
        stage('Scan the image')
        {
          steps{
                sh 'trivy image --format table -o trivy-image.html us-central1-docker.pkg.dev/project-1-449106/jenkins-project/boardgame:${BUILD_NUMBER} '
            }
            
       }
        
        stage('Push the image to GCR')
        {
            steps{
                withCredentials([file(credentialsId: 'gcrsvc-cred', variable: 'gcrsvc_cred')]) {
                    sh '''
                    gcloud auth list
                    gcloud auth configure-docker us-central1-docker.pkg.dev
                    
                    docker push us-central1-docker.pkg.dev/project-1-449106/jenkins-project/boardgame:${BUILD_NUMBER} 
                    '''
                }
            }
        }
        
        stage ('Deploy to K8s')
        {
            steps{
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://10.142.0.3:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        
        stage ('Verify the Deployment')
        {
            steps{
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://10.142.0.3:6443') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
        
    }
}
