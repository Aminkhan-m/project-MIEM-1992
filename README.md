# project-MIEM-1992
1992  Development of a workshop for conducting compositional analysis of software using the Code Scoring and Solar AppScreener tools.
1992-25-26/
├── Материалы проекта/
│   ├── Видеокурсы/
│   │   └── .gitkeep
│   ├── Исходный код/
│   │   └── sca_pipeline_v1.groovy
│   ├── Примеры отчётов анализа/
│   │   ├── Отчёт_анализа_data-prepper2.14_CodeScoring.pdf
│   │   └── Отчёт_анализа_data-prepper2.14_Solar_AppScreener.pdf
│   ├── Руководства/
│   │   ├── Документация_работы_sca_pipeline_v1.docx
│   │   └── Руководство_пользователя_по_использованию_продукта_CodeScoring.docx
│   ├── Учебные материалы/
│   │   ├── Методика_композиционного_анализа.docx
│   │   └── ПУД_1992_Версия_1.docx
│   └── Презентации/
│       └── 1992 представление.pptx
└── README.md

  Материалы проекта/Исходный код/sca_pipeline_v1.groovy  0 → 100644
+
131
−
0
pipeline {
    agent any

    options {
        timestamps()
    }

    parameters {
        string(
            name: 'GIT_REPO',
            defaultValue: 'student/your-repo.git',
            description: 'Путь до репозитория в Gitea (например, student/data_prepper_2-14.git)'
        )
        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Ветка для сканирования'
        )
        string(
            name: 'CODESCORING_PROJECT',
            defaultValue: 'student-project',
            description: 'Имя проекта в CodeScoring'
        )
        password(
            name: 'ANALYST_API_TOKEN',
            defaultValue: '',
            description: 'Персональный API-токен аналитика для CodeScoring. Если пусто — используется общий токен из Credentials Store.'
        )
        string(
            name: 'BUILDER_RESOLVE_FLAGS',
            defaultValue: '--pip-resolve --maven-resolve',
            description: 'Флаги резолверов сборщиков для johnny (например: --pip-resolve, --maven-resolve, --npm-resolve, --go-resolve)'
        )
        string(
            name: 'IGNORE_FLAGS',
            defaultValue: '--ignore .git --ignore .venv --ignore node_modules --ignore __pycache__ --ignore dist --ignore build --ignore target',
            description: 'Список путей, исключаемых из анализа (через --ignore)'
        )
    }

    environment {
        GITEA_HOST      = '172.16.1.3:3000'
        CODESCORING_URL = 'http://172.16.1.4:8081'
        WORK_DIR        = "scan-target-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Prepare workspace') {
            steps {
                sh 'mkdir -p "$WORK_DIR"'
            }
        }

        stage('Clone repository from Gitea') {
            steps {
                dir("${WORK_DIR}") {
                    withCredentials([usernamePassword(
                        credentialsId: 'gitea3-student-read-token',
                        usernameVariable: 'GITEA_USER',
                        passwordVariable: 'GITEA_TOKEN'
                    )]) {
                        sh '''
                            set -eu
                            git clone \
                              --branch "$GIT_BRANCH" \
                              "http://${GITEA_USER}:${GITEA_TOKEN}@${GITEA_HOST}/${GIT_REPO}" \
                              repo
                        '''
                    }
                }
            }
        }

        stage('Run Johnny scan') {
            steps {
                dir("${WORK_DIR}/repo") {
                    script {
                        def runScan = {
                            sh '''
                                set +e
                                command -v johnny
                                johnny scan dir . \
                                  --api_url "$CODESCORING_URL" \
                                  --api_token "$CODESCORING_API_TOKEN" \
                                  --save-results \
                                  --create-project \
                                  --project "$CODESCORING_PROJECT" \
                                  $BUILDER_RESOLVE_FLAGS \
                                  $IGNORE_FLAGS \
                                  --bom-path bom.json
                                rc=$?
                                echo "Johnny exit code: $rc"
                                if [ "$rc" -ne 0 ] && [ "$rc" -ne 1 ]; then
                                    exit "$rc"
                                fi
                                exit 0
                            '''
                        }

                        if (params.ANALYST_API_TOKEN?.trim()) {
                            echo 'Используется персональный токен аналитика.'
                            withEnv(["CODESCORING_API_TOKEN=${params.ANALYST_API_TOKEN}"]) {
                                runScan()
                            }
                        } else {
                            echo 'Используется общий токен из Jenkins Credentials Store.'
                            withCredentials([string(
                                credentialsId: 'codescoring-student-api-token',
                                variable: 'CODESCORING_API_TOKEN'
                            )]) {
                                runScan()
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: "scan-target-${env.BUILD_NUMBER}/repo/bom.json",
                allowEmptyArchive: true
            )
            dir("${WORK_DIR}") {
                deleteDir()
            }
