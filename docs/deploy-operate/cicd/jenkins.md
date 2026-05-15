---
title: Jenkins
---

# Jenkins

Automate build, test, and deployment of your WSO2 Integrator projects using Jenkins declarative pipelines.

## Overview

Jenkins provides a flexible CI/CD platform that can be self-hosted or run in the cloud. For WSO2 Integrator projects, a Jenkins pipeline compiles the Ballerina project, runs tests, builds a Docker image, and deploys to your target infrastructure. This guide uses a declarative `Jenkinsfile` checked into your repository.

## Prerequisites

- Jenkins server (version 2.387 or later) with the following plugins installed:
  - Pipeline plugin
  - Docker Pipeline plugin
  - Kubernetes CLI plugin (for K8s deployments)
  - JUnit plugin (for test result visualization)
- Java 17 installed on build agents
- Ballerina distribution installed on build agents (or installed during the pipeline)
- A container registry accessible from the Jenkins server
- Credentials configured in Jenkins for the registry and deployment target

## Pipeline configuration

Create a `Jenkinsfile` at the root of your repository:

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        BAL_VERSION     = '2201.10.0'
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME      = 'wso2-integrator-app'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        DOCKER_CREDS    = credentials('docker-registry-creds')
        KUBECONFIG      = credentials('kubeconfig-production')
    }

    tools {
        jdk 'JDK17'
    }

    stages {
        stage('Install Ballerina') {
            steps {
                sh '''
                    if ! command -v bal &> /dev/null; then
                        wget -q https://dist.ballerina.io/downloads/${BAL_VERSION}/ballerina-${BAL_VERSION}-swan-lake-linux-x64.deb
                        sudo dpkg -i ballerina-${BAL_VERSION}-swan-lake-linux-x64.deb
                    fi
                    bal version
                '''
            }
        }

        stage('Build') {
            steps {
                sh 'bal build'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/bin/*.jar', fingerprint: true
                }
            }
        }

        stage('Test') {
            steps {
                sh 'bal test --code-coverage'
            }
            post {
                always {
                    junit testResults: 'target/report/**/TEST-*.xml', allowEmptyResults: true
                    publishHTML(target: [
                        reportDir: 'target/report/test_results',
                        reportFiles: 'index.html',
                        reportName: 'Ballerina Test Report'
                    ])
                }
            }
        }

        stage('Docker Build & Push') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW} ${DOCKER_REGISTRY}
                    docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                                 -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest \
                                 -f target/docker/${IMAGE_NAME}/Dockerfile .
                    docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl --kubeconfig=${KUBECONFIG} set image \
                        deployment/${IMAGE_NAME} app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                        -n staging
                    kubectl --kubeconfig=${KUBECONFIG} rollout status \
                        deployment/${IMAGE_NAME} -n staging --timeout=300s
                """
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message 'Deploy to production?'
                ok 'Yes, deploy'
            }
            steps {
                sh """
                    kubectl --kubeconfig=${KUBECONFIG} set image \
                        deployment/${IMAGE_NAME} app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                        -n production
                    kubectl --kubeconfig=${KUBECONFIG} rollout status \
                        deployment/${IMAGE_NAME} -n production --timeout=300s
                """
            }
        }
    }

    post {
        failure {
            mail to: 'team@example.com',
                 subject: "Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Check the build at ${env.BUILD_URL}"
        }
        cleanup {
            cleanWs()
        }
    }
}
```

## Build stage details

The build stage compiles the Ballerina project and produces an executable JAR in `target/bin/`. When your project has a `Cloud.toml`, it also generates Docker and Kubernetes artifacts.

The `archiveArtifacts` step stores the built JAR in Jenkins for traceability and manual download.

## Test stage details

The test stage runs all test functions and publishes results in two formats:

- **JUnit XML** -- Displayed in the Jenkins build summary with pass/fail counts
- **HTML Report** -- A detailed test report viewable from the build page

```groovy
stage('Test') {
    steps {
        sh 'bal test --code-coverage'
    }
    post {
        always {
            junit testResults: 'target/report/**/TEST-*.xml', allowEmptyResults: true
        }
    }
}
```

## Deploy stage details

The pipeline deploys to staging automatically on every push to `main`. Production deployment requires manual approval via the `input` directive, which pauses the pipeline until an authorized user confirms.

For rollback scenarios, keep previous image tags available in your registry. Rolling back is a single `kubectl set image` command pointing to the prior tag.

## Secrets management

Use Jenkins Credentials to store sensitive values. Reference them in the pipeline using the `credentials()` helper:

| Credential ID | Type | Usage |
|---|---|---|
| `docker-registry-creds` | Username/Password | Container registry login |
| `kubeconfig-production` | Secret file | Kubernetes cluster access |
| `db-password` | Secret text | Database password |
| `api-key` | Secret text | External API key |

Pass secrets as environment variables to your Ballerina runtime:

```groovy
stage('Deploy') {
    environment {
        DB_PASSWORD = credentials('db-password')
    }
    steps {
        sh 'kubectl create secret generic app-secrets --from-literal=DB_PASSWORD=${DB_PASSWORD} -n production --dry-run=client -o yaml | kubectl apply -f -'
    }
}
```

## Multibranch pipeline

For teams using feature branches, configure a Jenkins Multibranch Pipeline that automatically discovers branches and runs the pipeline for each one. The `when` directives in the Jenkinsfile control which stages run for which branches:

- **Feature branches**: Build and Test only
- **Main branch**: Build, Test, Docker, and Deploy

## What's next

- [GitHub Actions](github-actions.md) -- CI/CD with GitHub-hosted runners
- [GitLab CI/CD](gitlab.md) -- Pipeline configuration for GitLab
- [Azure DevOps](azure-devops.md) -- Pipeline configuration for Azure DevOps
