pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "aniket1805/go-web-app:${env.BUILD_ID}"
  }

  options {
    skipStagesAfterUnstable()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set up Go') {
      tools {
        go 'Go-1.22'  // You must define this Go version under Jenkins tools
      }
    }

    stage('Build') {
      steps {
        sh 'go build -o go-web-app'
      }
    }

    stage('Test') {
      steps {
        sh 'go test ./...'
      }
    }

    stage('Code Quality') {
      steps {
        sh '''
          curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.56.2
          golangci-lint run
        '''
      }
    }

    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker build -t $DOCKER_IMAGE .
            docker push $DOCKER_IMAGE
          '''
        }
      }
    }

    stage('Update Helm Chart') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
          sh '''
            git config user.email "anikethulule7219@gmail.com"
            git config user.name "Aniket Hulule"
            sed -i 's/tag: .*/tag: "'$BUILD_ID'"/' helm/go-web-app-chart/values.yaml
            git add helm/go-web-app-chart/values.yaml
            git commit -m "Update tag in Helm chart"
            git push https://${GH_TOKEN}@github.com/<your-username>/<your-repo>.git HEAD:main
          '''
        }
      }
    }

  }

  post {
    failure {
      echo 'Pipeline failed.'
    }
  }
}
