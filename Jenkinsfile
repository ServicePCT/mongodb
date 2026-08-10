// mongodb — CI/CD (HAP-404, онбординг в новый Jenkins).
//
// Конфигурация сборки живёт ЗДЕСЬ, в репозитории проекта. Общие шаги — в библиотеке
// happydebt: https://github.com/ServicePCT/jenkins-shared-library
//
// Своей сборки и тестов у проекта нет: это готовый образ с конфигурацией. Поэтому
// pipeline короткий — проверка конфига, выкатка, проверка что стек поднялся.

@Library('happydebt') _

pipeline {
    agent any

    parameters {
        // ⚠️ При смене дефолта Jenkins применит новое значение со ВТОРОЙ сборки
        // (см. jenkins-infra/DEPLOY.md §10).
        string(name: 'STAGING_HOST', defaultValue: 'prog@vm-stage.happydebt.kz', description: 'user@host стенда; пусто — выкатка пропускается')
        string(name: 'STAGING_PATH', defaultValue: '/srv/mongodb', description: 'каталог с docker-compose.yml на стенде')
        string(name: 'PROD_HOST',    defaultValue: '', description: 'user@host прода; пусто — выкатка пропускается')
        string(name: 'PROD_PATH',    defaultValue: '/srv/mongodb', description: 'каталог с docker-compose.yml на проде')
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
        // Две выкатки базы одновременно — гонка на целевом хосте.
        disableConcurrentBuilds()
    }

    stages {
        stage('проверка конфигурации') {
            steps {
                // Своих тестов нет, но сломанный YAML лучше поймать здесь, чем на целевом
                // хосте в середине выкатки. Обязательных переменных в compose нет, поэтому
                // заглушки не нужны — значения по умолчанию подставятся сами.
                sh '''
                    set -eu
                    docker compose -f docker-compose.yml config >/dev/null
                    echo "docker-compose.yml валиден"
                '''
            }
        }

        stage('deploy staging') {
            when {
                allOf {
                    // Имя ветки по умолчанию не хардкодим — опираемся на флаг branch-api.
                    expression { env.BRANCH_IS_PRIMARY == 'true' }
                    expression { params.STAGING_HOST?.trim() }
                }
            }
            steps {
                composeDeploy(
                    env: 'staging',
                    host: params.STAGING_HOST,
                    path: params.STAGING_PATH,
                    approve: false
                )
            }
        }

        stage('deploy prod') {
            when {
                allOf {
                    expression { env.BRANCH_IS_PRIMARY == 'true' }
                    expression { params.PROD_HOST?.trim() }
                }
            }
            steps {
                // 🔴 Прод — только с явного подтверждения. Перезапуск базы означает, что
                // все ходящие в неё сервисы на это время остаются без данных.
                composeDeploy(
                    env: 'prod',
                    host: params.PROD_HOST,
                    path: params.PROD_PATH,
                    approve: true
                )
            }
        }
    }

    post {
        always  { notify(currentBuild.currentResult) }
        cleanup { cleanWs() }
    }
}
