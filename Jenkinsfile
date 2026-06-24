pipeline {

    agent any

    environment {
        CI          = 'true'
        EXPO_URL    = 'exp://192.168.1.100:8081'   // Substitua pelo IP real do host
        API_URL     = 'http://192.168.1.100:3000'  // Substitua pelo IP real do host
        ANDROID_AVD = 'Pixel_6_API_34'             // Nome do AVD criado no agente
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        // ─────────────────────────────────────────────
        // 1. Código
        // ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ─────────────────────────────────────────────
        // 2. Backend (Docker)
        // ─────────────────────────────────────────────
        stage('Start Backend') {
            steps {
                dir('ecommerce-fullstack') {
                    git url: 'https://github.com/jeanmalagi/ecommerce-fullstack.git',
                        branch: 'main'
                }
                bat '''
                cd ecommerce-fullstack
                docker compose down -v || exit 0
                docker compose up -d --build
                '''
            }
        }

        stage('Wait for API') {
            steps {
                bat '''
                @echo off
                setlocal EnableDelayedExpansion
                set RETRY=0
                :loop
                curl -s http://localhost:3000/api/products >nul 2>&1
                if !ERRORLEVEL!==0 (
                    echo API pronta
                    goto end
                )
                set /A RETRY+=1
                if !RETRY! GEQ 20 (
                    echo API nao subiu
                    exit /b 1
                )
                timeout /t 3 >nul
                goto loop
                :end
                '''
            }
        }

        // ─────────────────────────────────────────────
        // 3. App mobile (Expo)
        // ─────────────────────────────────────────────
        stage('Install App Dependencies') {
            steps {
                dir('ecommerce-mobile-app') {
                    git url: 'https://github.com/jeanmalagi/ecommerce-mobile-app.git',
                        branch: 'main'
                    bat 'npm install'
                }
            }
        }

        stage('Start Expo Server') {
            steps {
                // Inicia o servidor Expo em background e aguarda estar pronto
                bat '''
                cd ecommerce-mobile-app
                start /B npx expo start --port 8081 --no-dev --minify > ..\expo-server.log 2>&1
                '''
                bat '''
                @echo off
                setlocal EnableDelayedExpansion
                set RETRY=0
                :loop
                curl -s http://localhost:8081/status >nul 2>&1
                if !ERRORLEVEL!==0 (
                    echo Expo pronto
                    goto end
                )
                set /A RETRY+=1
                if !RETRY! GEQ 20 (
                    echo Expo nao subiu
                    exit /b 1
                )
                timeout /t 3 >nul
                goto loop
                :end
                '''
            }
        }

        // ─────────────────────────────────────────────
        // 4. Emulador Android
        // ─────────────────────────────────────────────
        stage('Start Android Emulator') {
            steps {
                bat '''
                @echo off
                echo Iniciando emulador %ANDROID_AVD%...
                start /B emulator -avd %ANDROID_AVD% -no-snapshot-load -no-audio -no-boot-anim

                REM Aguarda o emulator inicializar completamente
                set RETRY=0
                :loop
                adb shell getprop sys.boot_completed 2>nul | findstr "1" >nul
                if !ERRORLEVEL!==0 (
                    echo Emulador pronto
                    goto end
                )
                set /A RETRY+=1
                if !RETRY! GEQ 40 (
                    echo Emulador nao inicializou
                    exit /b 1
                )
                timeout /t 5 >nul
                goto loop
                :end
                adb shell input keyevent 82
                '''
            }
        }

        stage('Install Expo Go') {
            steps {
                // Instala o Expo Go no emulador caso ainda nao esteja instalado
                bat '''
                adb shell pm list packages | findstr "host.exp.exponent" >nul 2>&1
                if !ERRORLEVEL! NEQ 0 (
                    echo Baixando Expo Go...
                    curl -L -o expo-go.apk https://d1ahtucjixef4r.cloudfront.net/Exponent-2.31.3.apk
                    adb install expo-go.apk
                )
                echo Expo Go instalado
                '''
            }
        }

        // ─────────────────────────────────────────────
        // 5. Testes Maestro (paralelo por suite)
        // ─────────────────────────────────────────────
        stage('Tests') {

            failFast false

            parallel {

                stage('Auth - Login Admin') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/auth/login-admin.yaml --format junit --output test-results/login-admin.xml'
                        }
                    }
                }

                stage('Auth - Login Cliente') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/auth/login-cliente.yaml --format junit --output test-results/login-cliente.xml'
                        }
                    }
                }

                stage('Auth - Credenciais Invalidas') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/auth/login-credenciais-invalidas.yaml --format junit --output test-results/login-invalido.xml'
                        }
                    }
                }

                stage('Auth - Registro') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/auth/register.yaml --format junit --output test-results/register.xml'
                        }
                    }
                }

                stage('Customer - Browse Products') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/customer/browse-products.yaml --format junit --output test-results/browse-products.xml'
                        }
                    }
                }

                stage('Customer - Add to Cart') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/customer/add-to-cart.yaml --format junit --output test-results/add-to-cart.xml'
                        }
                    }
                }

                stage('Customer - Remove from Cart') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/customer/remove-from-cart.yaml --format junit --output test-results/remove-from-cart.xml'
                        }
                    }
                }

                stage('Customer - Checkout') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            bat 'maestro test .maestro/flows/customer/checkout.yaml --format junit --output test-results/checkout.xml'
                        }
                    }
                }
            }
        }
    }

    // ─────────────────────────────────────────────
    // Pós-build: relatórios e limpeza
    // ─────────────────────────────────────────────
    post {
        always {
            // Relatório JUnit nativo do Jenkins
            junit allowEmptyResults: true,
                  testResults: 'test-results/*.xml'

            // Artefatos de debug do Maestro
            archiveArtifacts artifacts: '.maestro/debug*/**/*',
                             allowEmptyArchive: true

            // Log do Expo
            archiveArtifacts artifacts: 'expo-server.log',
                             allowEmptyArchive: true

            // Para o emulador
            bat 'adb emu kill || exit 0'

            // Para o Docker
            bat '''
            cd ecommerce-fullstack
            docker compose down -v || exit 0
            '''
        }

        success {
            echo 'Todos os testes passaram com sucesso!'
        }

        unstable {
            echo 'Um ou mais testes falharam. Verifique o relatorio JUnit.'
        }

        failure {
            echo 'Pipeline falhou. Verifique os logs acima.'
        }
    }
}
