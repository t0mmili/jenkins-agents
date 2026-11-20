pipeline {
  agent { label 'docker-agent-general' }

  options {
    buildDiscarder logRotator(
      artifactDaysToKeepStr: '',
      artifactNumToKeepStr: '',
      daysToKeepStr: '',
      numToKeepStr: '5'
    )
    skipDefaultCheckout true
  }

  parameters {
    booleanParam(
      name: 'dryRun',
      defaultValue: true,
      description: 'Build and scan only. Nothing is pushed to Docker Hub.'
    )
    booleanParam(
      name: 'pushLatest',
      defaultValue: false,
      description: 'Tag as latest (ignored when dryRun is set)'
    )
    choice(
      name: 'agentImage',
      choices: ['docker', 'trivy'],
      description: 'Jenkins agent image to deploy'
    )
    string(
      name: 'dockerCredentials',
      defaultValue: 'docker-hub-creds',
      description: 'Jenkins credentials ID for Docker Hub',
      trim: true
    )
    string(
      name: 'dockerTag',
      defaultValue: '',
      description: 'Image tag',
      trim: true
    )
  }

  environment {
    AGENT_IMAGE = "${params.agentImage}"
    BUILDER     = "jenkins-builder-${env.BUILD_NUMBER}"
    CI_IMAGE    = "${params.agentImage == 'docker' ? 'jad-ci' : 'jat-ci'}:${env.BUILD_NUMBER}"
    DOCKER_REPO = "t0mmili/jenkins-inbound-agent-${params.agentImage}"
    IMAGE_TAG   = "${params.dockerTag}"
  }

  stages {
    stage('Pre-check') {
      agent any
      when {
        anyOf {
          equals expected: '', actual: dockerCredentials
          equals expected: '', actual: dockerTag
        }
      }
      steps {
        error 'One or more required job parameters are empty.'
      }
      post {
        cleanup {
          cleanWs()
        }
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Validate') {
      parallel {
        stage('Dockerfile lint') {
          steps {
            sh '''
              docker run \
                --rm -i \
                hadolint/hadolint \
                hadolint \
                --format json \
                --failure-threshold none \
                - < "Dockerfile.${AGENT_IMAGE}" \
                > "hadolint-results-${AGENT_IMAGE}.json"
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: 'hadolint-results-*.json', fingerprint: true
              recordIssues(
                enabledForFailure: true,
                qualityGates: [
                  [criticality: 'FAILURE', integerThreshold: 1, threshold: 1.0, type: 'NEW_ERROR'],
                  [criticality: 'FAILURE', integerThreshold: 1, threshold: 1.0, type: 'NEW_HIGH'],
                  [criticality: 'UNSTABLE', integerThreshold: 1, threshold: 1.0, type: 'NEW_NORMAL'],
                  [criticality: 'FAILURE', integerThreshold: 1, threshold: 1.0, type: 'TOTAL_ERROR']
                ],
                tools: [hadoLint(pattern: 'hadolint-results-*.json')]
              )
            }
          }
        }
      }
    }

    stage('Build images') {
      steps {
        sh '''
          docker buildx create \
            --name "${BUILDER}" \
            --driver docker-container \
            --driver-opt image=moby/buildkit:v0.15.1 \
            --bootstrap
        '''

        sh '''
          for platform in amd64 arm64; do
            docker buildx build \
              --builder "${BUILDER}" \
              --platform "linux/${platform}" \
              --build-arg IMAGE_VERSION="${IMAGE_TAG}" \
              --file "Dockerfile.${AGENT_IMAGE}" \
              --provenance=false \
              --sbom=false \
              --progress plain \
              --load \
              -t "${CI_IMAGE}-${platform}" \
              .
            docker save "${CI_IMAGE}-${platform}" -o "image-${platform}.tar"
          done
        '''
      }
      post {
        success {
          stash includes: 'image-*.tar', name: 'docker-images'
        }
      }
    }

    stage('Scan images') {
      agent { label 'docker-agent-trivy' }
      steps {
        unstash 'docker-images'

        sh '''
          for platform in amd64 arm64; do
            trivy image \
              --input "image-${platform}.tar" \
              --format json \
              --output "trivy-results-${AGENT_IMAGE}-${platform}.json" \
              --severity HIGH,CRITICAL \
              --exit-code 0 \
              --no-progress
          done
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-results-*.json', fingerprint: true
          recordIssues(
            enabledForFailure: true,
            qualityGates: [
              [criticality: 'FAILURE', integerThreshold: 1, type: 'NEW_ERROR'],
              [criticality: 'FAILURE', integerThreshold: 5, type: 'NEW_HIGH'],
              [criticality: 'UNSTABLE', integerThreshold: 1, type: 'TOTAL_ERROR']
            ],
            tools: [trivy(pattern: 'trivy-results-*.json')]
          )
        }
      }
    }

    stage('Publish') {
      when {
        expression { !params.dryRun }
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: dockerCredentials,
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
        }

        script {
          def tags = ["-t ${DOCKER_REPO}:${IMAGE_TAG}"]
          if (params.pushLatest) {
            tags << "-t ${DOCKER_REPO}:latest"
          }
          sh '''
            docker buildx build \
              --builder "${BUILDER}" \
              --platform "linux/amd64,linux/arm64" \
              --build-arg IMAGE_VERSION="${IMAGE_TAG}" \
              --file "Dockerfile.${AGENT_IMAGE}" \
              --provenance=true \
              --sbom=true \
              --progress plain \
              ${tags.join(' ')} \
              --push \
              .
          '''
        }

        sh 'docker buildx imagetools inspect "${DOCKER_REPO}:${IMAGE_TAG}"'
      }
    }
  }

  post {
    always {
      sh '''
        docker buildx rm "${BUILDER}" >/dev/null 2>&1 || true
        docker rmi -f "${CI_IMAGE}-amd64" "${CI_IMAGE}-arm64" >/dev/null 2>&1 || true
        docker logout || true
      '''
      cleanWs()
    }
    success {
      script {
        if (params.dryRun) {
          echo "Dry Run successful. Built and scanned ${params.dockerRepo}:${IMAGE_TAG}. Nothing published."
        } else {
          echo "Published successfully ${params.dockerRepo}:${IMAGE_TAG}"
        }
      }
    }
  }
}
