// Declarative pipeline. Needs plugins: Pipeline Utility Steps (readYaml), AWS Credentials.
pipeline {
  agent any
  options { timestamps(); disableConcurrentBuilds(); timeout(time: 4, unit: 'HOURS') }
  parameters {
    booleanParam(name: 'RUN_TRAINING', defaultValue: true, description: 'Retrain the model')
    booleanParam(name: 'DEPLOY_TO_PROD', defaultValue: false, description: 'Deploy after gate passes')
  }
  environment { AWS_DEFAULT_REGION = 'us-east-1' }   // keep in sync with config/pipeline.yaml

  stages {
    stage('Checkout & Setup') {
      steps {
        checkout scm
        script { env.CFG = readYaml(file: 'config/pipeline.yaml').project }
        sh '''
          python3 -m venv .venv && . .venv/bin/activate
          pip install --upgrade pip && pip install -r requirements.txt
        '''
      }
    }
    stage('Lint & Unit Tests') {
      steps {
        sh '''. .venv/bin/activate
              flake8 src pipeline monitoring --max-line-length=120
              pytest -q tests'''
      }
    }
    stage('Validate & Split Data') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-sagemaker-ci']]) {
          sh '. .venv/bin/activate && python pipeline/prepare_dataset.py'
        }
      }
    }
    stage('Train on SageMaker') {
      when { expression { params.RUN_TRAINING } }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-sagemaker-ci']]) {
          sh '. .venv/bin/activate && python pipeline/run_training.py'
        }
      }
    }
    stage('Quality Gate') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-sagemaker-ci']]) {
          sh '. .venv/bin/activate && python pipeline/evaluate_gate.py'   // non-zero exit = build fails
        }
      }
    }
    stage('Approve Production Deploy') {
      when { expression { params.DEPLOY_TO_PROD } }
      steps { input message: 'Gate passed. Deploy new model to production endpoint?', ok: 'Deploy' }
    }
    stage('Deploy Endpoint') {
      when { expression { params.DEPLOY_TO_PROD } }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-sagemaker-ci']]) {
          sh '. .venv/bin/activate && python pipeline/deploy_endpoint.py'
        }
      }
    }
    stage('Smoke Test & Monitoring') {
      when { expression { params.DEPLOY_TO_PROD } }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-sagemaker-ci']]) {
          sh '''. .venv/bin/activate
                python pipeline/smoke_test.py
                python monitoring/setup_cloudwatch.py'''
        }
      }
    }
  }
  post {
    failure { echo 'Pipeline failed - check the stage logs and CloudWatch.' }
    always  { archiveArtifacts artifacts: 'training_job.txt,model_data.txt', allowEmptyArchive: true }
  }
}
