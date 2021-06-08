pipeline{
    agent any
    tools{
        maven 'MAVEN'
    }
    environment {
    imageLine = 'farislinux/my-app-1.0:latest'
    }

    stages{
        stage('Compile Stage'){
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'farislinux', url: 'https://github.com/farislinux/jenkins-docker-example.git']]])
                sh 'mvn -Dmaven.test.failure.ignore=true clean package'
            }
        }
		stage('SonarQube Analysis') {
                steps {
                    withSonarQubeEnv('SonarQube') {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'mvn -Dsonar.test.exclusions=src/**/* sonar:sonar'
                        }
                    }
                }
            }
            stage("SonarQube Quality Gate") {
                steps {
                    timeout(time: 1, unit: 'HOURS') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        stage('Build Docker Image'){
            steps{
                script{
                   sh 'docker build -t farislinux/my-app-1.0 .' 
                }
                
            }
        }
        stage('Analyze with Anchore plugin') {
            steps {
            writeFile file: 'anchore_images', text: imageLine
            anchore bailOnFail: false, bailOnPluginFail: false, name: 'anchore_images'
          }
        }
        stage('Deploy Docker Image'){
            steps{
                script{
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                    sh 'docker login --username farislinux --password ${dockerhubpwd} '
                }
                sh 'docker push farislinux/my-app-1.0'
                }
            }			
        }
        stage('Deploy App on k8s') {
        steps {
            sshagent(['osboxes']) {
            sh "scp -o StrictHostKeyChecking=no ${WORKSPACE}/javapp.yaml osboxes@localhost:/home/osboxes"
            script {
                try{
                    sh "ssh osboxes@localhost kubectl apply -f javapp.yaml"
                }catch(error){
                    sh "ssh osboxes@localhost kubectl create -f javapp.yaml"
				}	
			}
        }
      
		}
	}
		stage('Clean') {
                steps{
                    script{
                        try{
                            sh 'docker rmi -f $(docker images -q -f dangling=true)'
                         } catch(Exception e){
                            echo 'No dangling images found. '
                         }
                     }
                }
        }
		
    }
}
