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
        // EMAIL_RECIPIENTS = 'eduardonaves41@gmail.com'
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
                        script {
                            def timestamp = new Date().format("yyyyMMdd-HHmmss", TimeZone.getTimeZone('UTC'))
                            def artifactFileName = "${env.ARTIFACT_NAME}-${timestamp}"
                            def artifactFullPath = "${env.ARTIFACT_PATH}"

                            echo "Artifact will be archived as: ${artifactFileName}"
                            archiveArtifacts artifacts: artifactFullPath, fingerprint: true
                        }
                    }
                }
            }
        }
    }

    post {
        // success {
        //     echo 'Pipeline finalizada com SUCESSO. Enviando notificação...'
        //     emailext(
        //         subject: "✅ SUCESSO: Pipeline ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
        //         body: "A pipeline ${env.JOB_NAME} foi concluída com sucesso.\nBuild URL: ${env.BUILD_URL}",
        //         recipientList: env.EMAIL_RECIPIENTS,
        //         attachLog: true
        //     )
        // }

        // failure {
        //     echo 'Pipeline finalizada com FALHA. Enviando notificação de erro...'
        //     emailext(
        //         subject: "❌ FALHA: Pipeline ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
        //         body: "ATENÇÃO: A pipeline ${env.JOB_NAME} FALHOU.\nPor favor, verifique o log em: ${env.BUILD_URL}/console",
        //         recipientList: env.EMAIL_RECIPIENTS,
        //         attachLog: true, // Anexa o log completo da build
        //         compressLog: true
        //     )
        // }

        always {
            echo 'Pipeline finalizado.'
        }
    }
}
