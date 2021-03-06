pipeline {
    agent any

    environment {
        proj="proj-4"
        py = "python"

        cred_id="yan-nagler"

        crs_dir = '%USERPROFILE%\\Desktop\\dev-ops-course'
        env_dir = "${crs_dir}\\env"
        py_dir = "${crs_dir}\\py"
        pkgs_dir = "${py_dir}\\venv\\Lib\\site-packages"

        dkr_img_base = "${proj}"
        dkr_reg_usr = "yannagler"

        dkr_img_reg_base = "${dkr_reg_usr}/${dkr_img_base}"
        dkr_img_reg = "${dkr_img_reg_base}:${BUILD_NUMBER}"
        dkr_img_reg_hndl = ""

        hlm_chart = "helm-chart"
        hlm_rls = "${proj}-${hlm_chart}"
        txt_k8s_svc_url = "k8s_url.txt"
        txt_k8s_svc_url_tmp = "k8s_url-tmp.txt"
    }

    stages {
        stage("Stage-1: Handle Git") {
            steps {
                script {
                    properties([
                        buildDiscarder(logRotator(
                                        artifactDaysToKeepStr: '',
                                        artifactNumToKeepStr: '',
                                        daysToKeepStr: '5',
//                                        numToKeepStr: '20')),
                                        numToKeepStr: '1')),

                        pipelineTriggers([pollSCM('30 * * * *')])
                    ])
                }

                git "https://github.com/yan-nagler-sw/${proj}.git"

                bat """
                    dir /A
                """
            }
        }

        stage("Stage-2: Handle prerequisites") {
            steps {
                echo "Running Python script: backend_testing_db.py..."
                bat """
                    set PYTHONPATH=%PYTHONPATH%;${pkgs_dir}
                    ${py} backend_testing_db.py
                """

                echo "Copying Selenium WebDriver - Chrome..."
                bat """
                    cp ${env_dir}/chromedriver .
                """
            }
        }

        stage("Stage-3: Launch REST server") {
            steps {
                echo "Running Python script: rest_app.py..."
                bat """
                    set PYTHONPATH=%PYTHONPATH%;${pkgs_dir}
                    start /min ${py} rest_app.py
                    sleep 5
                """
            }
        }

        stage("Stage-4: Launch BE test") {
            steps {
                echo "Running Python script: backend_testing.py..."
                bat """
                    set PYTHONPATH=%PYTHONPATH%;${pkgs_dir}
                    ${py} backend_testing.py
                """
            }
        }

        stage("Stage-5: Clean environment") {
            steps {
                echo "Running Python script: clean_environment.py..."
                bat """
                    set PYTHONPATH=%PYTHONPATH%;${pkgs_dir}
                    ${py} clean_environment.py
                """
            }
        }

        stage("Stage-6: Build Docker image") {
            steps {
                echo "Building Docker image: ${dkr_img_reg}..."
                script {
                    dkr_img_reg_hndl = docker.build dkr_img_reg
                }
                bat """
                    docker images
                """
            }
        }

        stage("Stage-7: Push Docker image to Hub") {
            steps {
                echo "Pushing Docker image to Hub..."
                script {
                    docker.withRegistry('', cred_id) {
                        dkr_img_reg_hndl.push()
                    }
                }
            }
        }

        stage("Stage-8: Set Docker image version") {
            steps {
                echo "Setting Docker image version: ${BUILD_NUMBER}..."
                bat """
                    echo IMAGE_TAG=${BUILD_NUMBER} > .env
                    type .env
                """
            }
        }

        stage("Stage-9: Build Docker container") {
            steps {
                echo "Build Docker container..."
                bat """
                    docker-compose up --build -d
                    sleep 5

                    docker ps -a
                """
            }
        }

        stage("Stage-10: Launch BE test - Docker") {
            steps {
                echo "Launching BE test - Docker: docker_backend_testing.py..."
                bat """
                    set PYTHONPATH=%PYTHONPATH%;${pkgs_dir}
                    ${py} docker_backend_testing.py
                """
            }
        }

        stage("Stage-11: Clean Docker environment") {
            steps {
                echo "Cleaning Docker environment..."
                bat """
                    docker-compose down
                    docker ps -a

                    docker rmi $dkr_img_reg
                    docker images
                """
            }
        }

        stage("Stage-12: Deploy Helm chart") {
            steps {
                echo "Deploying Helm chart..."
                bat """
                    helm install ${proj} --debug --set image.repository=${dkr_img_reg_base},image.tag=${BUILD_NUMBER} ${hlm_chart}
                    sleep 5

                    helm list --all
                """
            }
        }

        stage("Stage-13: Obtain K8S service URL") {
            steps {
                echo "Obtaining K8S service URL..."
            }
        }

        stage("Stage-14: Launch BE test - Helm") {
            steps {
                echo "Launching BE test - Helm: k8s_backend_testing.py..."
                bat """
                    start /min /b minikube service ${hlm_rls} --url > ${txt_k8s_svc_url_tmp}
                    sleep 10
                    (type ${txt_k8s_svc_url_tmp} | findstr "^http") > ${txt_k8s_svc_url}
                    type ${txt_k8s_svc_url}

                    set PYTHONPATH=%PYTHONPATH%;${pkgs_dir}
                    ${py} k8s_backend_testing.py
                """
            }
        }

        stage("Stage-15: Clean Helm environment") {
            steps {
                echo "Cleaning Helm environment..."
                bat """
                    rem helm delete ${proj}
                    helm list --all

                    del ${txt_k8s_svc_url_tmp}
                    del ${txt_k8s_svc_url}
                """
            }
        }
    }

    post {
        always {
            echo "post - always"
        }
        success {
            echo "post - success"
/*
            mail to: 'yan.nagler@gmail.com',
                 subject: "post - success: ${currentBuild.fullDisplayName}",
                 body: "post - success: ${env.BUILD_URL}"
*/
        }
        failure {
            echo "post - failure"
/*
            mail to: 'yan.nagler@gmail.com',
                 subject: "post - failure: ${currentBuild.fullDisplayName}",
                 body: "post - failure: ${env.BUILD_URL}"
*/
        }
        unstable {
            echo "post - unstable"
        }
        changed {
            echo "post - changed"
        }
    }
}
