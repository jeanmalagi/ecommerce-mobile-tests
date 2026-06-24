pipeline {

    agent any

    environment {
        CI          = 'true'
        EXPO_URL    = 'exp://192.168.1.100:8081'
        API_URL     = 'http://192.168.1.100:3000'
        ANDROID_AVD = 'Small_Phone'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Start Backend') {
            steps {

                dir('ecommerce-fullstack') {
                    git url: 'https://github.com/jeanmalagi/ecommerce-fullstack.git',
                        branch: 'main'
                }

                powershell '''
                $ErrorActionPreference = 'Stop'

                function Test-DockerReady {
                    try {
                        docker info *> $null
                        return $true
                    }
                    catch {
                        return $false
                    }
                }

                if (-not (Get-Command docker -ErrorAction SilentlyContinue)) {
                    throw "Docker CLI nao encontrado no agente Jenkins. Instale Docker e exponha o comando 'docker' no PATH."
                }

                if (-not (Test-DockerReady)) {
                    $dockerService = Get-Service -Name 'com.docker.service' -ErrorAction SilentlyContinue

                    if ($dockerService -and $dockerService.Status -ne 'Running') {
                        Write-Host "Servico do Docker encontrado, tentando iniciar..."
                        Start-Service -Name 'com.docker.service'

                        for ($i = 0; $i -lt 12; $i++) {
                            if (Test-DockerReady) {
                                break
                            }

                            Start-Sleep -Seconds 5
                        }
                    }
                }

                if (-not (Test-DockerReady)) {
                    throw "Docker daemon indisponivel para o agente Jenkins. O job precisa rodar em um no com Docker Engine acessivel ao usuario do servico Jenkins."
                }

                Push-Location 'ecommerce-fullstack'
                try {
                    docker compose down -v
                    docker compose up -d --build
                }
                finally {
                    Pop-Location
                }
                '''
            }
        }

        stage('Wait for API') {
            steps {
                powershell '''
                $maxRetries = 20

                for ($i = 0; $i -lt $maxRetries; $i++) {
                    try {
                        $response = Invoke-WebRequest `
                            -Uri "http://localhost:3000/api/products" `
                            -UseBasicParsing `
                            -TimeoutSec 5

                        if ($response.StatusCode -eq 200) {
                            Write-Host "API pronta"
                            exit 0
                        }
                    }
                    catch {
                        Write-Host "Aguardando API..."
                    }

                    Start-Sleep -Seconds 3
                }

                throw "API nao subiu"
                '''
            }
        }

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

                bat '''
                cd ecommerce-mobile-app

                start "Expo" cmd /c "npx expo start --port 8081 --no-dev --minify > ../expo-server.log 2>&1"
                '''

                bat '''
                @echo off
                setlocal EnableDelayedExpansion

                set RETRY=0

                :loop

                curl http://localhost:8081/status >nul 2>&1

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

        stage('Start Android Emulator') {
            steps {
                powershell '''
                $ErrorActionPreference = 'Stop'

                Write-Host "=========================================="
                Write-Host "Diagnostico Android"
                Write-Host "=========================================="
                whoami
                Write-Host "USERPROFILE=$env:USERPROFILE"
                Write-Host "HOME=$env:HOME"

                Write-Host "adb:"
                where.exe adb
                Write-Host "emulator:"
                where.exe emulator

                $emulatorCommand = Get-Command emulator -ErrorAction Stop
                $sdkRoot = Split-Path $emulatorCommand.Source -Parent | Split-Path -Parent

                $candidateAvdHomes = @()
                if ($env:ANDROID_AVD_HOME) {
                    $candidateAvdHomes += $env:ANDROID_AVD_HOME
                }

                if ($sdkRoot -match '^(?<userProfile>.+)\\AppData\\Local\\Android\\Sdk$') {
                    $candidateAvdHomes += (Join-Path $matches.userProfile '.android/avd')

                    if (-not $env:HOME) {
                        $env:HOME = $matches.userProfile
                    }
                }

                $candidateAvdHomes += (Join-Path $env:USERPROFILE '.android/avd')
                $candidateAvdHomes = @($candidateAvdHomes | Where-Object { $_ } | Select-Object -Unique)

                foreach ($candidateAvdHome in $candidateAvdHomes) {
                    if (Test-Path $candidateAvdHome) {
                        $env:ANDROID_AVD_HOME = $candidateAvdHome
                        Write-Host "ANDROID_AVD_HOME=$env:ANDROID_AVD_HOME"
                        break
                    }
                }

                $avds = @(emulator -list-avds | ForEach-Object { $_.Trim() } | Where-Object { $_ })

                if ($avds.Count -eq 0) {
                    throw "Nenhum AVD encontrado no agente Jenkins. Configure ANDROID_AVD_HOME para um diretorio existente ou execute o job em um usuario que tenha acesso aos AVDs."
                }

                $preferredAvd = $env:ANDROID_AVD
                if ($preferredAvd -and ($avds -contains $preferredAvd)) {
                    $avdToStart = $preferredAvd
                }
                else {
                    if ($preferredAvd) {
                        Write-Host "AVD '$preferredAvd' nao encontrado. Usando '$($avds[0])'."
                    }
                    $avdToStart = $avds[0]
                }

                Write-Host "=========================================="
                Write-Host "Iniciando emulador $avdToStart"
                Write-Host "=========================================="

                Start-Process -FilePath "emulator" -ArgumentList @(
                    "-avd", $avdToStart,
                    "-no-snapshot-load",
                    "-no-audio",
                    "-no-boot-anim",
                    "-no-window",
                    "-gpu", "swiftshader_indirect"
                ) | Out-Null

                adb wait-for-device | Out-Null

                $maxRetries = 60
                for ($i = 0; $i -lt $maxRetries; $i++) {
                    $boot = (adb shell getprop sys.boot_completed 2>$null).Trim()
                    Write-Host "Boot status: $boot"

                    if ($boot -eq '1') {
                        Write-Host "Emulador pronto"
                        adb shell input keyevent 82 | Out-Null
                        Write-Host "Emulador desbloqueado"
                        exit 0
                    }

                    Start-Sleep -Seconds 5
                }

                throw "Emulador nao inicializou"
                '''
            }
        }

        stage('Install Expo Go') {
            steps {

                bat '''
                @echo off
                setlocal EnableDelayedExpansion

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

        stage('Run Maestro Tests') {
            steps {

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/auth/login-admin.yaml --format junit --output test-results/login-admin.xml'
                }

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/auth/login-cliente.yaml --format junit --output test-results/login-cliente.xml'
                }

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/auth/login-credenciais-invalidas.yaml --format junit --output test-results/login-invalido.xml'
                }

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/auth/register.yaml --format junit --output test-results/register.xml'
                }

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/customer/browse-products.yaml --format junit --output test-results/browse-products.xml'
                }

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/customer/add-to-cart.yaml --format junit --output test-results/add-to-cart.xml'
                }

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/customer/remove-from-cart.yaml --format junit --output test-results/remove-from-cart.xml'
                }

                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    bat 'maestro test .maestro/flows/customer/checkout.yaml --format junit --output test-results/checkout.xml'
                }
            }
        }
    }

    post {

        always {

            junit allowEmptyResults: true,
                  testResults: 'test-results/*.xml'

            archiveArtifacts artifacts: '.maestro/debug*/**/*',
                             allowEmptyArchive: true

            archiveArtifacts artifacts: 'expo-server.log',
                             allowEmptyArchive: true

            bat '''
            adb emu kill
            '''

            bat '''
            cd ecommerce-fullstack
            docker compose down -v
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