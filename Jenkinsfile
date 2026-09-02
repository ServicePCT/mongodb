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

    environment {
        // Ключи выкатки — из хранилища ПАПКИ `mongo`, а не из общего (HAP-500). Общий
        // deploy-staging-ssh виден сборке любого подключённого проекта: сборка исполняет
        // Jenkinsfile из своего репозитория, и запросить чужой credential по id ей ничто
        // не мешает.
        // Своих секретов в Jenkins у проекта нет — в папке лежит только ключ выкатки. Смысл не в
        // защите секрета, а в том, чтобы перезапустить базу не могла сборка чужого сервиса.
        DEPLOY_CREDENTIALS_ID = 'deploy-mongo-staging-ssh'
        // Прод-ключа ещё нет — как и прод-хоста. Идентификатор проставлен заранее, чтобы
        // первая прод-выкатка упала с «credential not found», а не уехала на боевую машину
        // стендовым ключом.
        PROD_CREDENTIALS_ID   = 'deploy-mongo-prod-ssh'
    }

    parameters {
        // ⚠️ При смене дефолта Jenkins применит новое значение со ВТОРОЙ сборки
        // (см. jenkins-infra/DEPLOY.md §10).
        string(name: 'STAGING_HOST', defaultValue: 'prog@vm-stage.happydebt.kz', description: 'user@host стенда; пусто — выкатка пропускается')
        string(name: 'STAGING_PATH', defaultValue: '/srv/mongodb', description: 'каталог с docker-compose.yml на стенде')
        string(name: 'PROD_HOST',    defaultValue: '', description: 'user@host прода; пусто — выкатка пропускается')
        string(name: 'PROD_PATH',    defaultValue: '/srv/mongodb', description: 'каталог с docker-compose.yml на проде')
        // Выкатка по ВЕРСИИ (HAP-825). Клиентские инсталляции обновляются на тег релиза, а
        // не на main (правило 2 тиражирования, HAP-799). Отдельной джобы для тега нет: тег —
        // не ветка, и плодить по джобе на каждый релиз незачем. Версия приезжает параметром,
        // джоба берётся штатная, с основной ветки.
        //
        // Пусто — прежнее поведение целиком: собирается и катится коммит ветки.
        string(name: 'DEPLOY_REF', defaultValue: '',
               description: 'тег релиза (или коммит) для выкатки на клиентскую инсталляцию. Пусто — собранный коммит ветки. Контракт: jenkins-infra/docs/release-and-update.md')
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
        // disableConcurrentBuilds() убран (HAP-825). Он защищал по ДЖОБЕ, а гонка возможна
        // только по ЦЕЛИ — двум выкаткам в один каталог на одном хосте. Пока стенд был один,
        // разницы не было; с 10–15 клиентскими инсталляциями «клиент» — это значение
        // STAGING_HOST, и замок по джобе означает, что клиенты обновляются строго по очереди.
        // Замок по цели ("deploy:<host>:<path>") ставит сам composeDeploy: разные машины
        // обновляются параллельно, один каталог — по-прежнему по одному.
    }

    stages {
        stage('проверка конфигурации') {
            // Обновление клиента на УЖЕ ВЫПУЩЕННУЮ версию не пересобирает и не перепроверяет
            // её: тег проходил все гейты, когда его резали. Прогон здесь был бы не «лишней
            // проверкой», а проверкой ДРУГОГО кода — рабочая копия сборки стоит на основной
            // ветке, а выкатывается тег (HAP-825).
            when { expression { !params.DEPLOY_REF?.trim() } }
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
                    credentialsId: env.DEPLOY_CREDENTIALS_ID,
                    // Выкатка по версии (HAP-825): null => composeDeploy возьмёт env.GIT_COMMIT,
                    // то есть прежнее поведение. Непусто — на хосте будет checkout тега, и в лог
                    // уедет строка «выкачено: ref=… sha=… describe=…».
                    ref: params.DEPLOY_REF?.trim() ?: null,
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
                    credentialsId: env.PROD_CREDENTIALS_ID,
                    // Выкатка по версии (HAP-825): null => composeDeploy возьмёт env.GIT_COMMIT,
                    // то есть прежнее поведение. Непусто — на хосте будет checkout тега, и в лог
                    // уедет строка «выкачено: ref=… sha=… describe=…».
                    ref: params.DEPLOY_REF?.trim() ?: null,
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
