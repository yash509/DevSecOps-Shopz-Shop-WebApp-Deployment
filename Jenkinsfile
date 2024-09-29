pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment {
        IMAGE_NAME = "yash5090/countrybank"
        TAG = "${params.DOCKER_TAG}" 
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'jenkins --version'
                sh 'aws --version'
                sh 'kubectl version --client'
                sh 'terraform --version'
                sh 'trivy --version'
                sh 'docker --version'
                sh 'ansible --version'
                sh 'snyk --version'
            }
        }
        
        stage('Checkout from Git') {                        
            steps {                                       
                git branch: 'main', url: 'https://github.com/yash509/DevSecOps-CountryBank-WebApp-Deployment.git'
            }
        }
        
        stage('Deployments') {
            parallel {
                stage('Test deploy to staging') {
                    steps {
                        echo 'staging deployment done'
                    }
                }
                stage('Test deploy to production') {
                    steps {
                        echo 'production deployment done'
                    }
                }
            }
        }
        
        stage('Test Build') {
            steps {
                echo 'Building....'
            }
            post {
                always {
                    jiraSendBuildInfo site: 'clouddevopshunter.atlassian.net'
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to Staging from main....'
            }
            post {
                always {
                    jiraSendDeploymentInfo environmentId: 'us-stg-1', environmentName: 'us-stg-1', environmentType: 'staging', issueKeys: ['JIRA-1234']
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to Production from main....'
            }
            post {
                always {
                    jiraSendDeploymentInfo environmentId: 'us-prod-1', environmentName: 'us-prod-1', environmentType: 'production', issueKeys: ['JIRA-1234']
                }
            }
        }
        
        stage("SonarQube Code Analysis ") {                         
            steps {
                //dir('Band Website') {
                    withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=web-application \
                    -Dsonar.projectKey=web-application'''
                    //}
                }
            }
        }
        
        stage("Quality Gate") {
            steps {
                //dir('Band Website') {
                    script {
                        waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                    //}
                }
            }
        }
        
        stage('OWASP File System SCAN') {
            steps {
                //dir('Band Website') {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                //}
            }
        }
        
        stage('TRIVY File System SCAN') {
            steps {
                //dir('Band Website') {
                    sh "trivy fs . > trivyfs.txt"
                //}
            }
        }
        
        stage('Docker Scout Image Overview') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview' //docker scout cves local://breaking-bad:latest
                   }
                }   
            }
        }
        
        stage('Docker Scout CVES File System Scan') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout cves'
                   }
                }   
            }
        }

        stage('Snyk Loaded Vulnerability Test') {
        steps {
                withCredentials([string(credentialsId: 'snyk', variable: 'snyk')]) {
                   sh 'snyk auth $snyk'
                   sh 'snyk test --all-projects --report || true'
                   sh 'snyk code test --json-file-output=vuln1.json > snykloadedvulnerabilityreport.txt || true '
                }   
            }
        } 
        
        stage("Docker Image Building"){
            steps{
                script{
                    //dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                            sh "docker build -t ${IMAGE_NAME}:${TAG} ." 
                            
                        //}
                    }
                }
            }
        }
        
        stage("Docker Image Tagging"){
            steps{
                script{
                    //dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                            sh "docker tag ${IMAGE_NAME}:${TAG} ${IMAGE_NAME}:${TAG}" 
                        //}
                    }
                }
            }
        }
        
        stage('Docker Image Scanning') { 
            steps { 
                sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}:${TAG}" 
            } 
        } 
        
        stage("Image Push to DockerHub") {
            steps{
                script{
                    //dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            sh "docker push ${IMAGE_NAME}:${TAG}"
                        //}
                    }
                }
            }
        }
        
        stage('Docker Scout Image Scanning') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker-scout quickview ${IMAGE_NAME}:${TAG}"
                       sh "docker-scout cves ${IMAGE_NAME}:${TAG}"
                       sh "docker-scout recommendations ${IMAGE_NAME}:${TAG}"
                       sh "docker-scout attestation ${IMAGE_NAME}:${TAG}"
                   }
                }   
            }
        }

        stage('Snyk Docker Image Vulnerability Scannning') {
        steps {
                withCredentials([string(credentialsId: 'snyk', variable: 'snyk')]) {
                   sh 'snyk auth $snyk'
                   sh "snyk container test ${IMAGE_NAME}:${TAG} > snykvulnerabilityreport.txt --report || true"
                }
            }
        }    
        
        stage("TRIVY"){
            steps{
                //dir('Band Website') {
                    sh "trivy image ${IMAGE_NAME}:${TAG} > trivyimage.txt"   
                //}
            }
        }
        
        stage('Docker Scout Artifacts Analysis') {
            steps {
                script{
                  withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                      sh "docker-scout sbom --format list ${IMAGE_NAME}:${TAG}" // docker-scout sbom ${IMAGE_NAME}:${TAG} :latest
                    }
                }   
            }
        }
        
        stage ('Manual Approval'){
          steps {
           script {
             timeout(time: 10, unit: 'MINUTES') {
              approvalMailContent = """
              Project: ${env.JOB_NAME}
              Build Number: ${env.BUILD_NUMBER}
              Go to build URL and approve the deployment request.
              URL de build: ${env.BUILD_URL}
              """
             mail(
             to: 'clouddevopshunter@gmail.com',
             subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", 
             body: approvalMailContent,
             mimeType: 'text/plain'
             )
            input(
            id: "DeployGate",
            message: "Deploy ${params.project_name}?",
            submitter: "approver",
            parameters: [choice(name: 'action', choices: ['Deploy'], description: 'Approve deployment')])  
            }
           }
          }
        }

        stage ("Remove Docker Container") {
            steps{
                sh "docker stop countrybank | true"
                sh "docker rm countrybank | true"
             }
        }
        
        stage('Deploy to Docker Container'){
            steps{
                //dir('BMI Calculator (JS)') {
                    sh "docker run -d --name countrybank -p 3000:3000 ${IMAGE_NAME}:${TAG}" 
                //}
            }
        }

        stage ("Verify the Docker Deployments") {
            steps{
                sh "docker images -a"
                sh "docker ps -a"
             }
        }

        stage("Sanity Check for Shifting to Production") {
            steps {
                input "Should we ship to prod?"
            }
        }
        
        stage('Deploy to Kubernetes'){
            steps{
                script{
                    //dir('K8S') {
                        def deploymentFile = ""
                        if (params.DEPLOY_ENV == 'blue') {
                            deploymentFile = 'app-deployment-blue.yaml'
                        } else {
                            deploymentFile = 'app-deployment-green.yaml'
                        }
                        
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh "kubectl apply -f ${deploymentFile}"
                                sh 'kubectl apply -f service.yaml'
                                sh 'kubectl apply -f job.yaml'
                                sh 'kubectl apply -f pv.yaml'
                                sh 'kubectl apply -f pvc.yaml'
                                sh 'kubectl apply -f nwp.yaml'
                                sh 'kubectl apply -f ingress.yaml'
                        //}
                    }
                }
            }
        }

        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        sh '''
                            kubectl patch service countrybank-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"countrybank\\", \\"version\\": \\"''' + newEnv + '''\\"}}}"
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify the Kubernetes Deployments') { 
            steps { 
                withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') { 
                    sh "kubectl get all "
                    sh "kubectl get pods " 
                    sh "kubectl get svc "
                    sh "kubectl get ns"
                } 
            } 
        }
              
        stage('Deployment Done') {
            steps {
                echo 'Deployed Succcessfully...'
            }
        }
    }
    
    post { 
        always {
            script { 
                def jobName = env.JOB_NAME 
                def buildNumber = env.BUILD_NUMBER 
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN' 
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red' 
                def body = """ 
                <html> 
                <body> 
                <div style="border: 4px solid ${bannerColor}; padding: 10px;"> 
                <h2>${jobName} - Build ${buildNumber}</h2> 
                <div style="background-color: ${bannerColor}; padding: 10px;"> 
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3> 
                </div> 
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p> 
                </div> 
                </body> 
                </html> 
            """ 
 
            emailext (
                attachLog: true,
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}", 
                body: body, 
                to: 'clouddevopshunter@gmail.com', 
                from: 'jenkins@example.com', 
                replyTo: 'jenkins@example.com', 
                mimeType: 'text/html', 
                attachmentsPattern: 'trivy-image-report.html, trivyfs.txt, trivyimage.txt, snykvulnerabilityreport.txt, snykloadedvulnerabilityreport.txt') 
            } 
        } 
    }
}

stage('Result') {
        timeout(time: 10, unit: 'MINUTES') {
            mail to: 'clouddevopshunter@gmail.com',
            subject: "${currentBuild.result} CI: ${env.JOB_NAME}",
            body: "Project: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nGo to ${env.BUILD_URL} and approve deployment"
            input message: "Deploy ${params.project_name}?", 
            id: "DeployGate", 
            submitter: "approver", 
            parameters: [choice(name: 'action', choices: ['Success'], description: 'Approve deployment')]
        }
    }
