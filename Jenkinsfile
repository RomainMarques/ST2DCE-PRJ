pipeline {
    agent {
        label 'prj_agent1'
    }

    environment {
        DOCKER_IMAGE = '0'
        J_VERSION = '0'
        DOCKER_BUILD_ARGS = "--build-arg=VARIABLE=${BUILD_ID}"
        REPO_URL = 'https://github.com/RomainMarques/ST2DCE-PRJ'
    }

    stages {
        stage('Clone Repository') {
            steps {
                script {
                    git branch: 'main', url: 'https://github.com/RomainMarques/ST2DCE-PRJ.git'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image with version as Jenkins build ID
                    def pomFile = 'pom.xml'
                    def pom = readMavenPom file: pomFile
                    pom.version = "1.0.${BUILD_ID}-SNAPSHOT"
                    env.DOCKER_IMAGE = pom.artifactId
                    env.J_VERSION = pom.properties['java.version']
                    writeMavenPom file: pomFile, model: pom

                    //sh 'mvn versions:set -DnewVersion=st2dce-1.0.${BUILD_ID}-SNAPSHOT -f pom.xml'

                    dockerImage = docker.build("azarel9/prj_image:${J_VERSION}", "--build-arg VARIABLE=1.0.${BUILD_ID}-SNAPSHOT .")
                }
            }
        }

        stage('Publish Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerHubCred') {
                        dockerImage.push()
                    }
                }
            }
        }
        stage('Deploy in development') {
            steps {
                sh 'kubectl create namespace development'
                sh 'kubectl config set-context --current --namespace=development'
                script {
                    def current = sh(script: "kubectl config view --minify -o jsonpath='{..namespace}'", returnStdout: true).trim()
                    echo "Current kubectl context namespace: ${current}"
                }
                sh 'kubectl create deployment my-deployment --image=azarel9/prj_image:${J_VERSION} --replicas=2'
                sh 'minikube tunnel &'
                script {
                    def nbPod = 0
                    while (nbPod == 0) {
                        sh 'sleep 10'
                        nbPod = sh(script: 'kubectl get pods --selector=app=my-deployment --no-headers | wc -l', returnStdout: true).trim().toInteger()
                    }
                    echo 'Pod is now running'
                }
                sh 'kubectl expose deployment my-deployment --type=NodePort --port=8080'
                sh 'kubectl get pods'

                script {
                    def urlService = sh(script: "minikube service my-deployment -n development | grep -o 'http://[^ ]*'", returnStdout: true).trim()
                    sh "curl ${urlService}"
                }
            }
        }
        stage('Check Number of replicas in development') {
            steps {
                script {
                    def nbPods = sh(script: 'kubectl get pods --selector=app=my-deployment --no-headers | wc -l', returnStdout: true).trim().toInteger()
                    assert nbPods >= 2, "Number of my app pods (${nbPods}) is less than 2"
                }
            }
        }
        stage('Delete pods development and recreate them with yaml') {
            steps {
                sh 'kubectl delete deployments my-deployment'
                sh 'kubectl delete svc my-deployment'
                sh 'envsubst < my-deployment.yaml | kubectl apply -f -'
                sh 'sleep 10'
            }
        }
        stage('Check Number of replicas in development with yaml files') {
            steps {
                script {
                    def nbPods = sh(script: 'kubectl get pods --selector=app=my-deployment --no-headers | wc -l', returnStdout: true).trim().toInteger()
                    assert nbPods >= 2, "Number of my app pods (${nbPods}) is less than 2"
                }
            }
        }
        stage('Deploy in production') {
            steps {
                sh 'kubectl create namespace production'
                sh 'kubectl config set-context --current --namespace=production'
                script {
                    def current = sh(script: "kubectl config view --minify -o jsonpath='{..namespace}'", returnStdout: true).trim()
                    echo "Current kubectl context namespace: ${current}"
                }
                sh 'kubectl create deployment my-prod --image=azarel9/prj_image:${J_VERSION} --replicas=2'
                script {
                    def nbPod = 0
                    while (nbPod == 0) {
                        sh 'sleep 10'
                        nbPod = sh(script: 'kubectl get pods --selector=app=my-prod --no-headers | wc -l', returnStdout: true).trim().toInteger()
                    }
                    echo 'Pod is now running'
                }
                sh 'kubectl expose deployment my-prod --type=NodePort --port=8080'
                sh 'kubectl get pods'

                script {
                    def urlService = sh(script: "minikube service my-prod -n production | grep -o 'http://[^ ]*'", returnStdout: true).trim()
                    sh "curl ${urlService}"
                }
            }
        }
        stage('Check Number of replicas in production') {
            steps {
                script {
                    def nbPods = sh(script: 'kubectl get pods --selector=app=my-prod --no-headers | wc -l', returnStdout: true).trim().toInteger()
                    assert nbPods >= 2, "Number of my app pods (${nbPods}) is less than 2"
                }
            }
        }
        stage('Delete pods production and recreate them with yaml') {
            steps {
                sh 'kubectl delete deployments my-prod'
                sh 'kubectl delete svc my-prod'
                sh 'envsubst < my-deployment.yaml | kubectl apply -f -'
                sh 'sleep 10'
            }
        }
        stage('Check Number of replicas in production with yaml files') {
            steps {
                script {
                    def nbPods = sh(script: 'kubectl get pods --selector=app=my-deployment --no-headers | wc -l', returnStdout: true).trim().toInteger()
                    assert nbPods >= 2, "Number of my app pods (${nbPods}) is less than 2"
                }
            }
        }
    }
    post {
        always {
            sh 'kubectl delete namespace development'
            sh 'kubectl delete namespace production'
        }
    }
}
