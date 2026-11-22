pipeline {
  agent any
  environment {
    IMAGE = "ranjanmr1989/demo-app"
    TAG   = "${BUILD_NUMBER}"
  }
  stages {
    stage('Build') {
      steps {
        sh 'docker build -t $IMAGE:$TAG app/'
      }
    }
    stage('Security scan (Trivy)') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --exit-code 1 --severity CRITICAL --ignore-unfixed $IMAGE:$TAG
        '''
      }
    }
    stage('Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub',
            usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
          sh '''
            echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$TAG
          '''
        }
      }
    }
    stage('Update manifest (GitOps)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github',
            usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh '''
            git config user.email "jenkins@devops-lab"
            git config user.name "jenkins"
            sed -i "s|^  repository:.*|  repository: $IMAGE|" charts/demo-app/values.yaml
            sed -i "s|^  tag:.*|  tag: \\"$TAG\\"|" charts/demo-app/values.yaml
            git add charts/demo-app/values.yaml
            git commit -m "ci: deploy $IMAGE:$TAG"
            git push https://$GH_USER:$GH_TOKEN@github.com/$GH_USER/devops-lab.git HEAD:main
          '''
        }
      }
    }
  }
}
