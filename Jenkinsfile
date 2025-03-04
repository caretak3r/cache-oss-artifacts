pipeline {
    agent any
    parameters {
        choice(name: 'PROJECT_NAME', choices: getChartNames(), description: 'Select Helm chart project')
        choice(name: 'ACTION', choices: ['sync', 'use_existing'], description: 'Sync to latest or use existing version')
        choice(name: 'VERSION', choices: getVersions(params.PROJECT_NAME), description: 'Select version (if using existing)', required: false)
    }
    environment {
        ARTIFACTORY_HELM_REPO = 'https://artifactory.example.com/artifactory/helm-local'
        ECR_REGISTRY = '123456789012.dkr.ecr.us-east-1.amazonaws.com'
        HELM_EXPERIMENTAL_OCI = 1
    }
    stages {
        stage('Sync Version') {
            when { expression { params.ACTION == 'sync' } }
            steps {
                script {
                    def chartInfo = getChartInfo(params.PROJECT_NAME)
                    def latestVersion = getLatestVersion(chartInfo.repo, chartInfo.chartName)
                    def currentVersions = chartInfo.versions
                    
                    if (!currentVersions.contains(latestVersion)) {
                        echo "Discovered new version: ${latestVersion}"
                        addVersionToManifest(params.PROJECT_NAME, latestVersion)
                        env.CHART_VERSION = latestVersion
                    } else {
                        echo "Already have latest version ${latestVersion}"
                        env.CHART_VERSION = latestVersion
                    }
                    
                    env.HELM_REPO_URL = chartInfo.repo
                    env.HELM_CHART_NAME = chartInfo.chartName
                }
            }
        }
        stage('Resolve Existing Version') {
            when { expression { params.ACTION == 'use_existing' } }
            steps {
                script {
                    if (!params.VERSION) {
                        error("Version must be selected when using existing versions")
                    }
                    def chartInfo = getChartInfo(params.PROJECT_NAME)
                    env.CHART_VERSION = params.VERSION
                    env.HELM_REPO_URL = chartInfo.repo
                    env.HELM_CHART_NAME = chartInfo.chartName
                }
            }
        }
        stage('Fetch & Cache Helm Chart') {
            steps {
                script {
                    helmRepoAdd()
                    sh """
                        helm fetch ${env.HELM_CHART_NAME} --version ${env.CHART_VERSION} --untar
                        helm image pull ./${env.HELM_CHART_NAME}-${env.CHART_VERSION}
                    """
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
        stage('Push Images to ECR') {
            steps {
                script {
                    def images = sh(script: "helm image export ./${env.HELM_CHART_NAME}-${env.CHART_VERSION} | grep 'image:' | awk '{print \$2}' | sort -u", returnStdout: true).trim().split('\n')
                    
                    images.each { image ->
                        dockerLoginECR()
                        def ecrImage = "${env.ECR_REGISTRY}/${image.split('/').last()}"
                        sh """
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

def getLatestVersion(String repoUrl, String chartName) {
    helmRepoAddTemp(repoUrl)
    def versions = sh(script: "helm search repo ${chartName} --versions --output json | jq -r '.[0].version'", returnStdout: true).trim()
    return versions
}

def helmRepoAdd() {
    def chartInfo = getChartInfo(params.PROJECT_NAME)
    sh """
        helm repo add ${params.PROJECT_NAME} ${chartInfo.repo}
        helm repo update
    """
}

def helmRepoAddTemp(String repoUrl) {
    sh """
        helm repo add temp-repo ${repoUrl}
        helm repo update
    """
}

def addVersionToManifest(String projectName, String newVersion) {
    def manifest = readJSON file: 'manifest.json'
    def chart = manifest.charts.find { it.name == projectName }
    if (!chart.versions.contains(newVersion)) {
        chart.versions.add(newVersion)
        writeJSON file: 'manifest.json', json: manifest, pretty: 4
    }
}

def dockerLoginECR() {
    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
        sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}"
    }
}
