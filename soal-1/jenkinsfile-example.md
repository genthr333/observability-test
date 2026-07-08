# Contoh Kerangka Jenkinsfile — DevSecOps Pipeline

> Ini kerangka konseptual untuk presentasi, sesuaikan nama tool/registry dengan environment aktual.

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = "registry.internal/myapp"
        IMAGE_TAG  = "${env.GIT_COMMIT}"
    }

    stages {

        stage('Secret Scanning') {
            steps {
                sh 'gitleaks detect --source=. --exit-code=1'
            }
        }

        stage('SAST') {
            steps {
                sh 'semgrep --config=auto --error .'
                // atau: sonar-scanner (kalau pakai SonarQube)
            }
        }

        stage('Dependency Scan (SCA)') {
            steps {
                sh 'trivy fs --severity CRITICAL,HIGH --exit-code 1 .'
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Image Scan') {
            steps {
                sh "trivy image --severity CRITICAL,HIGH --exit-code 1 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Push Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh "helm upgrade --install myapp ./charts/myapp -f values-staging.yaml --set image.tag=${IMAGE_TAG} -n staging"
            }
        }

        stage('DAST') {
            steps {
                sh 'zap-baseline.py -t https://staging.myapp.internal -r zap-report.html'
            }
        }

        stage('Approval') {
            steps {
                input message: 'Deploy ke Production?', ok: 'Deploy'
            }
        }

        stage('Deploy to Production') {
            steps {
                sh "helm upgrade --install myapp ./charts/myapp -f values-prod.yaml --set image.tag=${IMAGE_TAG} -n production"
            }
        }
    }

    post {
        failure {
            // notifikasi pasif ke tim
            sh 'curl -X POST $SLACK_WEBHOOK -d "Pipeline gagal: ${JOB_NAME} #${BUILD_NUMBER}"'
        }
    }
}
```

**Poin yang perlu ditekankan saat presentasi:**
- Stage security ada **sebelum** build image → fail fast, hemat resource.
- Threshold severity (`CRITICAL,HIGH`) bisa disesuaikan kebijakan organisasi.
- Ada gate manual (`input`) sebelum production — bukan full-auto, karena masih perlu human judgement di titik kritis.
- `post { failure { ... } }` adalah contoh notifikasi pasif otomatis.
