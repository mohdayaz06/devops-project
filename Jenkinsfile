pipeline {
    agent {
        docker {
        image 'whatever is suitable for entire pipeline'
        args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        AWS_REGION    = 'ap-south-1'
        ECR_REPO      = '593241949985.dkr.ecr.ap-south-1.amazonaws.com/recommendationservice'
        SERVICE_DIR  = 'src/recommendation'
        DOCKERFILE   = 'src/recommendation/Dockerfile'
        IMAGE_TAG     = "${env.BUILD_NUMBER}"
        IMAGE_URI     = "${ECR_REPO}:${IMAGE_TAG}"
        K8S_MANIFEST  = 'kubernetes/recommendation/deploy.yaml'
    }

    stages {
        stage( 'Checkout' ) {
            steps {
                checkout scm
            }
        }   

/*       stage ( 'set up python') {
            steps {
                sh '''
                    cd ${SERVICE_DIR}
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            We can ignore this stage if we already have these dependencies installed in the docker image used as agent
            }
       } 
*/
         stage ( 'Unit Tests' ) {
                steps {
                    sh '''
                        apk add --no-cache python3 py3-pip # install python3 and pip if not already available in the docker image
                        cd ${SERVICE_DIR}
                        pip3 install -r requirements.txt
                        pytest -q || python3 -m unittest -q # if unit test fails, the build will stop here automatically.
                    '''
                }
         }

         stage ( 'Build Docker Image' ) {
            steps {
                sh '''
                    cd ${SERVICE_DIR}
                    docker build -t ${IMAGE_URI} -f Dockerfile .
                '''
            }
         }

        stage('Login to ECR') {
            steps {
                withCredentials([
                [$class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-creds']
                ]) {
                    sh '''
                    aws ecr get-login-password --region $AWS_REGION \
                    | docker login --username AWS --password-stdin 593241949985.dkr.ecr.ap-south-1.amazonaws.com
                    '''
                }
            }
        }

            stage ( 'Push Image to ECR' ) {
                steps {
                    sh '''
                        docker push ${IMAGE_URI}
                    '''
                }
            }

            stage ( 'Update K8s Deployment' ) {
                steps {
                    sh '''
                        sed -i 's|image: .*|image: '${IMAGE_URI}'|g' ${K8S_MANIFEST}
                    '''
                }
            }

/*            stage ( 'Commit and Push K8s Manifest Changes' ) {
                steps {
                    withCredentials([usernamePassword(credentialsId: 'github-creds', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh '''
                            git config --global user.email "<EMAIL>"
                            git config --global user.name "Jenkins"
                            git add ${K8S_MANIFEST}
                            git commit -m "Update image to ${IMAGE_URI}"
                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/your-username/your-repo.git HEAD:main
                        '''
                        we can ignore this stage if we don't want to commit the manifest changes back to repo, we generally do this only if
                        we have a separate repo for k8s manifests and want to keep it updated and also in case of GitOps practices.
                    }
                }
            }
*/
            stage ( 'Deploy to EKS and Rollout Status' ) {
                steps {
                    sh '''
                        aws eks --region $AWS_REGION update-kubeconfig --name my-eks-cluster
                        cd kubernetes/recommendation
                        kubectl apply -f deploy.yaml
                        sleep 20
                        kubectl rollout status deployment/opentelemetry-demo-recommendationservice

                        #Rollback can be done using below command if needed
                        #kubectl rollout undo deployment/opentelemetry-demo-recommendationservice
                    '''
                }
            }
    }
}