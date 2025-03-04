pipeline {
    agent any
    parameters {
        choice(name: 'CHART_NAME', choices: getChartNames(), description: 'Select Helm chart from manifest.json')
        choice(name: 'VERSION_SOURCE', choices: ['listed', 'custom'], description: 'Choose version from list or enter custom')
        choice(name: 'LISTED_VERSION', choices: getVersions(params.CHART_NAME), description: 'Select version', required: false)
        string(name: 'CUSTOM_VERSION', defaultValue: '', description: 'Enter custom version', required: false)
    }
    environment {
        ARTIFACTORY_HELM_REPO = 'https://artifactory.example.com/artifactory/helm-local'
        ECR_REGISTRY = '123456789012.dkr.ecr.us-east-1.amazonaws.com'
        HELM_EXPERIMENTAL_OCI = 1
    }
    stages {
        stage('Resolve Version') {
            steps {
                script {
                    env.CHART_VERSION = (params.VERSION_SOURCE == 'listed') ? params.LISTED_VERSION : params.CUSTOM_VERSION
                    if (!env.CHART_VERSION?.trim()) {
                        error("Chart version must be specified")
                    }
                    def chartInfo = getChartInfo(params.CHART_NAME)
                    env.HELM_REPO_URL = chartInfo.repo
                    env.HELM_CHART_NAME = chartInfo.chartName
                }
            }
        }
        stage('Fetch & Cache Helm Chart') {
            steps {
                script {
                    helmRepoAdd()
                    sh "helm fetch ${env.HELM_CHART_NAME} --version ${env.CHART_VERSION} --untar"
                }
            }
        }
        stage('Push Helm Chart to Artifactory') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'artifactory-creds', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh """
                        helm package ${env.HELM_CHART_NAME}-${env.CHART_VERSION} -d ./
                        curl -u ${ARTIFACTORY_USER}:${ARTIFACTORY_PASSWORD} -T ${env.HELM_CHART_NAME}-${env.CHART_VERSION}.tgz "${env.ARTIFACTORY_HELM_REPO}/"
                    """
                }
            }
        }
        stage('Extract & Push Images to ECR') {
            steps {
                script {
                    def images = sh(script: "helm template ${env.HELM_CHART_NAME}-${env.CHART_VERSION} | grep 'image:' | awk '{print \$2}' | tr -d '\"' | sort -u", returnStdout: true).trim().split('\n')
                    images.each { image ->
                        dockerLoginECR()
                        def ecrImage = "${env.ECR_REGISTRY}/${image.split('/').last()}"
                        sh """
                            docker pull ${image}
                            docker tag ${image} ${ecrImage}
                            docker push ${ecrImage}
                        """
                    }
                }
            }
        }
    }
}

// Helper functions
def getChartNames() {
    def manifest = readJSON file: 'manifest.json'
    return manifest.charts.collect { it.name }
}

def getVersions(String chartName) {
    def chart = getChartInfo(chartName)
    return chart.versions
}

def getChartInfo(String chartName) {
    def manifest = readJSON file: 'manifest.json'
    return manifest.charts.find { it.name == chartName }
}

def helmRepoAdd() {
    sh """
        helm repo add ${env.CHART_NAME} ${env.HELM_REPO_URL}
        helm repo update
    """
}

def dockerLoginECR() {
    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
        sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}"
    }
}
