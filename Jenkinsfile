pipeline {
    agent any

    environment {
        proj="proj-3"
        py = "python"

        crs_dir = '%USERPROFILE%\\Desktop\\dev-ops-course'
        env_dir = "${crs_dir}\\env"
        py_dir = "${crs_dir}\\py"
        pkgs_dir = "${py_dir}\\venv\\Lib\\site-packages"

        dkr_img_name = "$proj"
        dkr_repo_usr = "yannagler"
        dkr_img_name_repo = "${dkr_repo_usr}/${dkr_img_name}"

        dkr_svc = "rest"
        dkr_img_name_cmp = "${dkr_img_name}_${dkr_svc}"

        img_ver = "1.2.3"
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
                                        numToKeepStr: '20')),

                        pipelineTriggers([pollSCM('30 * * * *')])
                    ])
                }

                git 'https://github.com/yan-nagler-sw/proj-3.git'

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
                echo "Building Docker image: ${dkr_img_name}..."
                bat """
                    docker build -t ${dkr_img_name} .
                    docker images
                """
            }
        }

        stage("Stage-7: Push Docker image to Hub") {
            steps {
                echo "Pushing Docker image to Hub: ${dkr_img_name} -> ${dkr_img_name_repo}..."
                bat """
                    docker login
                    docker tag ${dkr_img_name} ${dkr_img_name_repo}
                    docker push ${dkr_img_name_repo}
                """
            }
        }

        stage("Stage-8: Set compose image version") {
            steps {
                echo "Setting compose image version: ${img_ver}..."
                bat """
                    echo IMAGE_TAG=${img_ver} > .env
                    type .env
                """
            }
        }

        stage("Stage-9: Build Docker container") {
            steps {
                echo "Build Docker container..."
                bat """
                    docker-compose up --build -d
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
    }

    post {
        always {
            echo "post - always"

            echo "Cleaning Docker environment..."
            bat """
                docker-compose down

                docker rmi -f ${dkr_img_name_repo}
                docker rmi -f ${dkr_img_name}
                docker rmi -f ${dkr_img_name_cmp}

                docker ps -a
                docker images
            """
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
