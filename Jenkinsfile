def CONTAINER_NAME = "calculator" //nom du container
def ENV_NAME = getEnvName(env.BRANCH_NAME)//nom de la branche et l'environnement dans lequel nous souhaitons la déterminé
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME) //détermine le tag de notre container qui est stable  et qui instable en envrironnement préprod c-a-d UAT et DEV
def HTTP_PORT = getHTTPPort(env.BRANCH_NAME)// sappuie sur le nom de la branche pour déterminé le numéro port de lenvironnemment considéré prod ,uat ou dev
def EMAIL_RECIPIENTS = "abdoulayediankha44@gmail.com"//recoie les message demail par rapport aux echecs et aux succes


node {
    try {
        stage('Initialize')//premiére étape déclarer toutes les outils dont nous aurons besoin
        {
            def dockerHome = tool 'DockerLatest' //Maintenir le meme nom que celui du docker définit dans jenkins
            def mavenHome = tool 'MavenLatest'   //Maintenir le meme nom que celui du maven définit dans jenkins
            env.PATH = "${dockerHome}/bin:${mavenHome}/bin:${env.PATH}"//Rajout de lintallation docker et maven dans les variables denvironnement pour y avoir acces de nimporte comme si on etait dans un systeme
        }

        stage('Checkout') {
            checkout scm //Commande jenkins qui est utiliser pour pouvoir aller taper dans le repository git et recuperer toutes les branches qui ont ete pusher sur git et voir sil ya un changement et dans le caas ou il ya des changements on lexpression ci dessous
        }

        stage('Build with test') {

            sh "mvn clean install" //dans le caas ou il ya des changements on fera cette expression qui va lancer les tests. Si les tests passent il va construire le package que nous allons utiliser pour pouvoir demarer limage pour pouvoir contuire limage docker
        }

        stage('Sonarqube Analysis') {
            withSonarQubeEnv('SonarQubeLocalServer') {
                sh " mvn sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"//sonar fait son analyse et si le projet est ok il va contacter jenkins pour lui dire que le qualitygatestatus est ok
            }
            timeout(time: 1, unit: 'MINUTES') { //wait un temps maximum dune minute
                def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv wait le retour de sonarqube
                if (qg.status != 'OK') {//dans le cas ou le status nest pas ok on arrete le pipeline
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }

        stage("Image Prune") {
            imagePrune(CONTAINER_NAME)
        }

        stage('Image Build') {
            imageBuild(CONTAINER_NAME, CONTAINER_TAG)
        }

        stage('Push to Docker Registry') {
            withCredentials([usernamePassword(credentialsId: 'DockerhubCredentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                pushToImage(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD)//Pusher notre image vers dockerhub
            }
        }

        stage('Run App') {
            withCredentials([usernamePassword(credentialsId: 'DockerhubCredentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                runApp(CONTAINER_NAME, CONTAINER_TAG, USERNAME, HTTP_PORT, ENV_NAME) //execute my app

            }
        }

    } finally {
        deleteDir()
        sendEmail(EMAIL_RECIPIENTS);
    }

}

def imagePrune(containerName) {
    try {
        sh "docker image prune -f"
        sh "docker stop $containerName"
    } catch (ignored) {
    }
}

def imageBuild(containerName, tag) {
    sh "docker build -t $containerName:$tag  -t $containerName --pull --no-cache ."
    echo "Image build complete"
}

def pushToImage(containerName, tag, dockerUser, dockerPassword) {
    sh "docker login -u $dockerUser -p $dockerPassword"
    sh "docker tag $containerName:$tag $dockerUser/$containerName:$tag"
    sh "docker push $dockerUser/$containerName:$tag"
    echo "Image push complete"
}

def runApp(containerName, tag, dockerHubUser, httpPort, envName) {
    sh "docker pull $dockerHubUser/$containerName"
    sh "docker run --rm --env SPRING_ACTIVE_PROFILES=$envName -d -p $httpPort:$httpPort --name $containerName $dockerHubUser/$containerName:$tag"
    echo "Application started on port: ${httpPort} (http)"
}

def sendEmail(recipients) {
    mail(
            to: recipients,
            subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult} - (${currentBuild.fullDisplayName})",
            body: "Check console output at: ${env.BUILD_URL}/console" + "\n")
}

String getEnvName(String branchName) {
    if (branchName == 'main') {
        return 'prod'
    }
    return (branchName == 'ready') ? 'uat' : 'dev'
}

String getHTTPPort(String branchName) {
    if (branchName == 'main') {
        return '9999'
    }
    return (branchName == 'ready') ? '8888' : '8090'
}

String getTag(String buildNumber, String branchName) {
    if (branchName == 'main') {
        return buildNumber + '-stable'
    }
    return buildNumber + '-unstable'
}
