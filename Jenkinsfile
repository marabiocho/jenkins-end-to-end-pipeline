
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            app: test
        spec:
          containers:
          - name: maven
            image: maven:3.8.3-adoptopenjdk-11
            command:
            - cat
            tty: true
            volumeMounts:
            - mountPath: "/root/.m2/repository"
              name: cache
          - name: git
            image: bitnami/git:latest
            command:
            - cat
            tty: true
          - name: docker
            image: docker:latest
            command:
            - cat
            tty: true
            volumeMounts:
            - mountPath: /var/run/docker.sock
              name: docker-sock
          - name: sonarcli
            image: sonarsource/sonar-scanner-cli:latest
            command:
            - cat
            tty: true
          - name: kubectl-helm-cli
            image: kunchalavikram/kubectl_helm_cli:latest
            command:
            - cat
            tty: true
          - name: curl
            image: alpine/curl:latest
            command:
            - cat
            tty: true
          volumes:
          - name: cache
            persistentVolumeClaim:
              claimName: maven-cache
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
      '''
    }      
  }
  environment{
    NEXUS_VERSION = "nexus3"
    NEXUS_PROTOCOL = "http"
    NEXUS_URL = "143.244.211.161:8081"
    NEXUS_REPOSITORY = "maven-hosted"
    NEXUS_CREDENTIAL_ID = "nexus-creds"
    DOCKERHUB_USERNAME = "kantin10"
    APP_NAME = "spring-petclinic"
    IMAGE_NAME = "${DOCKERHUB_USERNAME}" + "/" + "${APP_NAME}"
    IMAGE_TAG = "${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout SCM') {
      when { expression { true } }
      steps {
        container('git') {
          git url: 'https://github.com/marabiocho/spring-petclinic.git',
          branch: 'main'
        }
      }
      
    }
    stage('Build SW'){
      when { expression { true } }
      steps {
        container('maven'){
          sh 'mvn -Dmaven.test.failure.ignore=true clean package'
        }
      }
      
    }
    stage('Sonar Scan'){
      when { expression { true } }
      steps{
        container('sonarcli'){
          withSonarQubeEnv(credentialsId: 'sonar', installationName: 'sonarserver') { 
            sh '''/opt/sonar-scanner/bin/sonar-scanner \
              -Dsonar.projectKey=petclinic \
              -Dsonar.projectName=petclinic \
              -Dsonar.projectVersion=1.0 \
              -Dsonar.sources=src/main \
              -Dsonar.tests=src/test \
              -Dsonar.java.binaries=target/classes  \
              -Dsonar.language=java \
              -Dsonar.sourceEncoding=UTF-8 \
              -Dsonar.java.libraries=target/classes
            '''
          }
        }
      }
      
    }
    stage('Wait for Quality Gate'){
      when { expression { false } }
      steps{
        container('sonarcli'){
          timeout(time: 1, unit: 'HOURS') {
            waitForQualityGate abortPipeline: true
          }
        }
      }
      
    }
    stage('Publish Maven Artifacts to Nexus'){
      when { expression { true } }
      steps {
        container('jnlp'){
          script {
            pom = readMavenPom file: "pom.xml";
            filesByGlob = findFiles(glob: "target/*.${pom.packaging}"); 
            echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
            artifactPath = filesByGlob[0].path;
            artifactExists = fileExists artifactPath;
            if(artifactExists) {
                echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                nexusArtifactUploader(
                    nexusVersion: NEXUS_VERSION,
                    protocol: NEXUS_PROTOCOL,
                    nexusUrl: NEXUS_URL,
                    groupId: pom.groupId,
                    version: pom.version,
                    repository: NEXUS_REPOSITORY,
                    credentialsId: NEXUS_CREDENTIAL_ID,
                    artifacts: [
                        [artifactId: pom.artifactId,
                        classifier: '',
                        file: artifactPath,
                        type: pom.packaging],

                        [artifactId: pom.artifactId,
                        classifier: '',
                        file: "pom.xml",
                        type: "pom"]
                    ]
                );

            } else {
                error "*** File: ${artifactPath}, could not be found";
            }
          }
        }
      }
      
    }
    
    stage('Build Docker Image'){
      when { expression { true } }
      steps{
        container('docker'){
          sh "docker build -t $IMAGE_NAME:$IMAGE_TAG ."
          sh "docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest"

        }

      }
    }

    stage('Docker push') {
      when { expression { true } }
        steps {
           container('docker'){
             sh '''
             docker login -u kantin10 -p Kantin10@
             docker push $IMAGE_NAME:$IMAGE_TAG
             docker push $IMAGE_NAME:latest
          
             docker rmi $IMAGE_NAME:$IMAGE_TAG
            docker rmi $IMAGE_NAME:latest

            '''
        }
      }
      
    }

    stage("Deploy to Kubernetes via manifests"){
      when { expression { false} }
      steps{
        container('kubectl-helm-cli'){
          withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', serverUrl: '') {
            sh '''
            
            kubectl apply -f deployment.yml
            '''
          }
        }
      }
    }

    stage("Deploy to Kubernetes via Helm"){
      when { expression { true } }
      steps{
        container('kubectl-helm-cli'){
          withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', serverUrl: '') {
            sh "helm upgrade --install petclinic petclinic-chart/"
          }
        }
      }
      
    }

  }

post {
    failure {
      mail to: 'kantinstephanetongue@gmail.com',
      from: 'jenkinsadmin@gmail.com',
      subject: "Jenkins pipeline has failed for job ${env.JOB_NAME}",
      body: "Check build logs at ${env.BUILD_URL}"
    }
    success {
      mail to: 'kantinstephanetongue@gmail.com',
      from: 'jenkinsadmin@gmail.com',
      subject: "Jenkins pipeline for job ${env.JOB_NAME} is completed successfully",
      body: "Check build logs at ${env.BUILD_URL}"
    }
  }

}
