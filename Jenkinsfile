pipeline {
  agent any

  parameters {
    string(name: 'repo', defaultValue: '', description: 'Repository name')
    string(name: 'owner', defaultValue: '', description: 'Owner of the repo')
    string(name: 'branch', defaultValue: 'main', description: 'Branch name to clone')
  }

  environment {
    AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    AWS_DEFAULT_REGION = 'us-west-2'
    S3_BUCKET = 'cloudsmith-artifacts'
    DEPLOYMENT_API_URL = 'https://hajmm6mslc.execute-api.us-west-2.amazonaws.com/default/update-stage' // your API endpoint
  }

  stages {
    stage('Initialize Deployment') {
      steps {
        echo "🛠 Initializing deployment stages in DynamoDB"
        sh """
        curl -X POST $DEPLOYMENT_API_URL \
        -H "Content-Type: application/json" \
        -d '{
          "repoName": "${params.owner}/${params.repo}",
          "updates": [
            { "stageName": "GitPulse Event Emitted", "status": "pending" },
            { "stageName": "BuildRelay Triggered Jenkins", "status": "pending" },
            { "stageName": "Jenkins Cloned Repo", "status": "pending" },
            { "stageName": "Uploaded to S3", "status": "pending" },
            { "stageName": "Lambda Saved Artifact", "status": "pending" }
          ]
        }'
        """
      }
    }
    
    stage('Clone Repository') {
      steps {
        echo "📦 Cloning https://github.com/${params.owner}/${params.repo}.git (branch: ${params.branch})"
        git branch: "${params.branch}", url: "https://github.com/${params.owner}/${params.repo}.git"
      }
    }

    stage('Zip Build') {
      steps {
        sh 'zip -r artifact.zip .'
        sh """
        curl -X POST $DEPLOYMENT_API_URL \
        -H "Content-Type: application/json" \
        -d '{
        "repoName": "${params.owner}/${params.repo}",
        "updates": [
            { "stageName": "GitPulse Event Emitted", "status": "done" },
            { "stageName": "BuildRelay Triggered Jenkins", "status": "done" },
            { "stageName": "Jenkins Cloned Repo", "status": "done" }
        ]
        }'
        """
      }
    }

    stage('Upload to S3') {
      steps {
        sh 'aws s3 cp artifact.zip s3://$S3_BUCKET/$BUILD_ID/artifact.zip'
      }
    }

    stage('Emit Artifact Uploaded Event') {
      steps {
        sh """
        ARTIFACT_URL=https://${S3_BUCKET}.s3.amazonaws.com/\${BUILD_ID}/artifact.zip
        curl -X POST https://n9vaozeoqe.execute-api.us-west-2.amazonaws.com/default/artifact-uploaded \
        -H "Content-Type: application/json" \
        -d '{
            "repo": "${params.owner}/${params.repo}",
            "artifactUrl": "'\${ARTIFACT_URL}'",
            "buildId": "${BUILD_ID}",
            "timestamp": "'\$(date --utc +%FT%TZ)'"
        }'
        """
      }
    }

    stage('Print Artifact URL') {
      steps {
        script {
          def url = "https://${env.S3_BUCKET}.s3.amazonaws.com/${env.BUILD_ID}/artifact.zip"
          echo "✅ Artifact uploaded: ${url}"
        }
      }
    }
  }
}