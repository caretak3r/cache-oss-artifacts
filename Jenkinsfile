pipeline {
    agent any
    environment {
        ECR_REGISTRY = '123456789012.dkr.ecr.us-east-1.amazonaws.com'
        HELM_EXPERIMENTAL_OCI = 1
    }
    stages {
        stage('Process Updates') {
            steps {
                script {
                    def manifest = readJSON file: 'manifest.json'
                    
                    manifest.charts.each { chart ->
                        def chartUpdates = processChart(chart)
                        if(chartUpdates) {
                            updateManifest(chart, chartUpdates)
                        }
                    }
                    
                    writeJSON file: 'manifest.json', json: manifest, pretty: 4
                }
            }
        }
    }
}

def processChart(chart) {
    def updates = []
    try {
        def repoUrl = chart.repo
        def chartName = chart.chartName
        
        // Get sorted semantic versions from repo
        def remoteVersions = getHelmVersions(repoUrl, chartName)
        if(!remoteVersions) return []
        
        // Get latest local version
        def localVersions = chart.versions ? chart.versions.collect { it.helmVersion } : []
        def latestLocal = localVersions ? localVersions.max() : '0.0.0'
        
        // Find new versions needing processing
        def newVersions = remoteVersions.findAll { compareVersions(it, latestLocal) > 0 }
        
        newVersions.each { version ->
            def images = []
            dir(version) {
                // Fetch and extract chart data
                helmFetch(repoUrl, chartName, version)
                images = extractImages(chartName, version)
                
                // Push images to ECR
                pushImagesToECR(images)
            }
            
            updates << [
                helmVersion: version,
                images: images.collect { [name: it.name, version: it.tag] }
            ]
        }
    } catch(ex) {
        echo "Failed processing ${chart.name}: ${ex}"
    }
    return updates
}

// Helper functions
def getHelmVersions(String repoUrl, String chartName) {
    helmRepoAddTemp(repoUrl)
    def versions = sh(
        script: "helm search repo ${chartName} --versions --output json | jq -r '.[] | select(.name == \"${chartName}\") | .version'",
        returnStdout: true
    ).trim().split('\n')
    return versions.sort { a, b -> compareVersions(b, a) } // Descending order
}

def compareVersions(String a, String b) {
    def partsA = a.split('[.-]').collect { it.isInteger() ? it.toInteger() : it }
    def partsB = b.split('[.-]').collect { it.isInteger() ? it.toInteger() : it }
    def maxLength = Math.max(partsA.size(), partsB.size())
    
    for(int i=0; i<maxLength; i++) {
        def partA = i < partsA.size() ? partsA[i] : 0
        def partB = i < partsB.size() ? partsB[i] : 0
        
        if(partA != partB) {
            return partA <=> partB
        }
    }
    return 0
}

def helmFetch(String repoUrl, String chartName, String version) {
    sh """
        helm repo add temp-repo ${repoUrl}
        helm repo update
        helm fetch temp-repo/${chartName} --version ${version} --untar
        helm image pull ./${chartName}-${version}
    """
}

def extractImages(String chartName, String version) {
    def images = sh(
        script: """
            helm image export ./${chartName}-${version} | \
            grep 'image:' | \
            awk '{print \$2}' | \
            sort -u
        """, 
        returnStdout: true
    ).trim().split('\n')
    
    return images.collect { img ->
        def parts = img.split(':')
        [name: parts[0], tag: parts.size() > 1 ? parts[1] : 'latest']
    }
}

def pushImagesToECR(images) {
    dockerLoginECR()
    
    images.each { img ->
        def ecrImage = "${env.ECR_REGISTRY}/${img.name.split('/').last()}"
        sh """
            docker pull ${img.name}:${img.tag}
            docker tag ${img.name}:${img.tag} ${ecrImage}:${img.tag}
            docker push ${ecrImage}:${img.tag}
        """
    }
}

def updateManifest(chart, updates) {
    if(!chart.versions) chart.versions = []
    chart.versions.addAll(updates)
}

def dockerLoginECR() {
    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
        sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}"
    }
}
