pipeline {
  agent any

  tools {
    go 'go-test'
  }

  environment {
    DOCKER_IMAGE = "aniket1805/go-web-app:${BUILD_ID}"
    NEXUS_URL = "http://3.91.208.177:8081/repository/reports"
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
          curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b /var/lib/jenkins/go/bin v1.56.2
          export PATH=$PATH:/var/lib/jenkins/go/bin
          golangci-lint run --out-format json > golangci-lint-report.json
        '''
      }
    }

    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh """
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker build -t $DOCKER_IMAGE .
            docker push $DOCKER_IMAGE
          """
        }
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh "trivy image --format json --output trivy-report.json $DOCKER_IMAGE"
      }
    }

    stage('Update Helm Chart') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
          sh """
            git config user.email "anikethulule7219@gmail.com"
            git config user.name "Aniket Hulule"
            sed -i 's/tag: .*/tag: "${BUILD_ID}"/' helm/go-web-app-chart/values.yaml
            git add helm/go-web-app-chart/values.yaml
            git commit -m "Update tag in Helm chart"
            git push https://${GITHUB_TOKEN}@github.com/anikethulule2000/Go-web-app-cicd-project.git HEAD:main
          """
        }
      }
    }

    stage('Upload Reports to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh """
            echo "Listing report files before upload:"
            ls -lh *.json || echo "No JSON reports found."

            curl -v -u $NEXUS_USER:$NEXUS_PASS \
              --upload-file golangci-lint-report.json \
              ${NEXUS_URL}/golangci-lint-report-${BUILD_ID}.json

            curl -v -u $NEXUS_USER:$NEXUS_PASS \
              --upload-file trivy-report.json \
              ${NEXUS_URL}/trivy-report-${BUILD_ID}.json
          """
        }
      }
    }
  }

  post {
    always {
      echo 'Archiving generated reports...'
      archiveArtifacts artifacts: '*.json', allowEmptyArchive: true
    }

    failure {
      echo '❌ Pipeline failed.'
    }

    success {
      echo '✅ Pipeline completed successfully.'
    }
  }
}
