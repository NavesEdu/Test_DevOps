pipeline {
    agent { label 'cpp-agent' }

    parameters {
        booleanParam(name: 'SKIP_CHECKS_AND_TESTS', defaultValue: false, description: 'Marque esta opção para um build rápido que apenas gera o artefato, pulando as verificações de código e os testes.')
        booleanParam(name: 'GENERATE_ARTIFACT', defaultValue: false, description: 'Marque para gerar artefato manualmente sem esperar o cron.')
    }

    triggers {
        cron('H 0 * * *') 
    }

    environment {
        ARTIFACT_NAME = 'calculator'
        ARTIFACT_PATH = "calculator/src/bin/${ARTIFACT_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            stages {
                stage('Code Check') {
                    when { expression { return params.SKIP_CHECKS_AND_TESTS == false } }
                    steps {
                        dir('calculator') {
                            sh 'make check'
                        }
                    }
                }

                stage('Build') {
                    steps {
                        dir('calculator') {
                            sh 'make'
                        }
                    }
                }

                stage('Test') {
                    when { expression { return params.SKIP_CHECKS_AND_TESTS == false } }
                    steps {
                        dir('calculator') {
                            sh 'make unittest'
                        }
                    }
                }

                stage('Archive Artifacts') {
                    when { expression { return params.GENERATE_ARTIFACT || currentBuild.getBuildCauses('hudson.triggers.TimerTrigger$TimerTriggerCause') } }
                    steps {
                        def timestamp = new Date().format("yyyyMMdd-HHmmss", TimeZone.getTimeZone('UTC'))
                            def artifactFileName = "${env.ARTIFACT_NAME}-${timestamp}"
                            def artifactFullPath = "${env.ARTIFACT_PATH}"

                            archiveArtifacts artifacts: artifactFullPath, fingerprint: true
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finalizado.'
        }
    }
}
