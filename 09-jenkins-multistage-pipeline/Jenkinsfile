pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Creating virtual environment and installing dependencies...'
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'python3 -m unittest discover -s .'
            }
        }
        stage('Code Quality') {
            steps {
                echo 'Running code quality checks...'
                sh 'flake8 --exclude=python-app-deploy .'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                sh '''
                mkdir -p ${WORKSPACE}/python-app-deploy
                cp ${WORKSPACE}/app.py ${WORKSPACE}/python-app-deploy/
                '''
            }
        }
        stage('Run Application') {
            steps {
                echo 'Running application...'
                sh '''
                nohup python3 ${WORKSPACE}/python-app-deploy/app.py > ${WORKSPACE}/python-app-deploy/app.log 2>&1 &
                echo $! > ${WORKSPACE}/python-app-deploy/app.pid
                '''
            }
        }
        stage('Test Application') {
            steps {
                echo 'Testing application...'
                sh '''
                python3 ${WORKSPACE}/test_app.py
                '''
            }
        }
        stage('Docker Deploy') {
            steps {
                echo 'Building and running Docker container...'
                sh '''
                docker build -t flask-pipeline-app .
                docker rm -f flask-pipeline-app || true
                docker run -d --name flask-pipeline-app -p 5001:5000 flask-pipeline-app
                docker ps --filter name=flask-pipeline-app
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
            mail to: 'team@example.com',
                 subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Pipeline succeeded. Details: ${env.BUILD_URL}"
        }
        failure {
            echo 'Pipeline failed. Check the logs for more details.'
            mail to: 'team@example.com',
                 subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Pipeline failed. Check the logs: ${env.BUILD_URL}"
        }
    }
}
