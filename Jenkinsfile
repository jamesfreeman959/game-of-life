node('docker') {
    checkout scm
    echo "1. PWD: ${pwd()}"

    stage 'Build Web App'
    docker.image('cloudbees/java-build-tools:0.0.5').inside {
        echo "2. PWD: ${pwd()}"
        sh "mvn -B -V -Dmaven.local.repo=${pwd()}/.m2/repo clean package"
    }

    // build docker image 'game-of-life' and push it to ECR
    stage 'Build & Push Docker Image'

    echo 'Build docker image game-of-life...'
    sh 'docker build -t 564007293907.dkr.ecr.us-east-1.amazonaws.com/game-of-life:latest gameoflife-web'

    echo 'Push docker image game-of-life to ECR...'
    docker.withRegistry('https://564007293907.dkr.ecr.us-east-1.amazonaws.com', 'ecr:aws') {
        sh 'docker push 564007293907.dkr.ecr.us-east-1.amazonaws.com/game-of-life:latest'
    }

    stage 'Redeploy ECS Service'
    wrap([$class: 'AmazonAwsCliBuildWrapper', credentialsId: 'aws', defaultRegion: 'us-east-1']) {
        // TODO THESE ARE PROBABLY NOT THE BEST ECS CALLS
        sh "aws ecs update-service --service game-of-life --desired-count 0"
        sleep 60
        sh "aws ecs update-service --service game-of-life --desired-count 1"
        sleep 20
    }

    stage 'Web Browser tests'
    mail body: "Start web browser tests on http://gameoflife-ecs.beesshop.org/ ?",subject: "Start web browser tests on http://gameoflife-ecs.beesshop.org/ ?", to: 'cleclerc@cloudbees.com'

    input "Start web browser tests on http://gameoflife-ecs.beesshop.org/ ?"

    // web browser tests are fragile, test up to 3 times
    retry(3) {
        docker.image('cloudbees/java-build-tools:0.0.5').inside {
            echo "3. PWD: ${pwd()}"
            sh """
               curl http://gameoflife-ecs.beesshop.org/
               cd gameoflife-acceptance-tests
               mvn -B -V -s -Dmaven.local.repo=${pwd()}/.m2/repo verify -Dwebdriver.driver=remote -Dwebdriver.base.url=http://gameoflife-ecs.beesshop.org/
            """
        }
    }
}
