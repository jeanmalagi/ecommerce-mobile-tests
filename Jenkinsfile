pipeline {

    agent any

    environment {
        CI                  = 'true'
        EXPO_URL            = 'exp://10.0.2.2:8081/--/'
        API_URL             = 'http://10.0.2.2:3000/api'
        EXPO_PUBLIC_API_URL = 'http://10.0.2.2:3000/api'
        ANDROID_AVD         = 'Small_Phone'
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

        stage('Prepare Test Environment') {
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
                    $overrideCompose = @"
services:
  backend:
    environment:
      JWT_SECRET: jenkins-mobile-tests-secret
"@

                    Set-Content -Path 'docker-compose.jenkins.yml' -Value $overrideCompose -Encoding ASCII

                    docker compose down -v
                    docker compose -f docker-compose.yml -f docker-compose.jenkins.yml up -d --build
                }
                finally {
                    Pop-Location
                }
                '''

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

                powershell '''
                $ErrorActionPreference = 'Stop'

                function Ensure-User {
                    param(
                        [string]$Name,
                        [string]$Email,
                        [string]$Password
                    )

                    $payload = @{
                        name = $Name
                        email = $Email
                        password = $Password
                    } | ConvertTo-Json

                    try {
                        Invoke-RestMethod -Method Post -Uri 'http://localhost:3000/api/users/register' -ContentType 'application/json' -Body $payload -TimeoutSec 10 | Out-Null

                        Write-Host "Usuario criado: $Email"
                    }
                    catch {
                        $response = $_.Exception.Response
                        if ($response -and [int]$response.StatusCode -eq 400) {
                            Write-Host "Usuario ja existe: $Email"
                        }
                        else {
                            throw
                        }
                    }
                }

                Ensure-User -Name 'Admin CI' -Email 'admin@email.com' -Password '123456'
                Ensure-User -Name 'Cliente CI' -Email 'cliente@email.com' -Password '123456'

                Push-Location 'ecommerce-fullstack'
                try {
                    docker compose exec -T db psql -U postgres -d ecommerce -c "UPDATE users SET is_admin = true WHERE email = 'admin@email.com';"
                }
                finally {
                    Pop-Location
                }
                '''

                dir('ecommerce-mobile-app') {

                    git url: 'https://github.com/jeanmalagi/ecommerce-mobile-app.git',
                        branch: 'main'

                    bat 'npm install'
                }
            }
        }

        stage('Prepare Mobile Runtime') {
            steps {

                bat '''
                @echo off
                setlocal EnableDelayedExpansion

                for /f "tokens=5" %%p in ('netstat -ano ^| findstr ":8081" ^| findstr "LISTENING"') do (
                    echo Encerrando processo na porta 8081 ^(PID %%p^)...
                    taskkill /PID %%p /F >nul 2>&1
                )

                exit /b 0
                '''

                bat '''
                cd ecommerce-mobile-app

                start "Expo" cmd /c "set EXPO_PUBLIC_API_URL=%EXPO_PUBLIC_API_URL%&& echo EXPO_PUBLIC_API_URL=%EXPO_PUBLIC_API_URL% > ../expo-server.log && npx expo start --clear --port 8081 --no-dev --minify >> ../expo-server.log 2>&1"

                exit /b 0
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

                timeout /t 3 /nobreak >nul

                goto loop

                :end
                '''

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

                $sdkOwnerProfile = Split-Path (Split-Path (Split-Path (Split-Path $sdkRoot -Parent) -Parent) -Parent) -Parent
                if ($sdkOwnerProfile -and (Test-Path $sdkOwnerProfile)) {
                    $candidateAvdHomes += (Join-Path $sdkOwnerProfile '.android/avd')

                    if (-not $env:HOME) {
                        $env:HOME = $sdkOwnerProfile
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

                adb wait-for-device 2>$null | Out-Null

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

                powershell '''
                $ErrorActionPreference = 'Stop'

                Write-Host "=========================================="
                Write-Host "Validando emulador ativo"
                Write-Host "=========================================="

                adb start-server | Out-Null

                $emuSerial = $null
                $maxRetries = 24

                for ($i = 0; $i -lt $maxRetries; $i++) {
                    $deviceLines = @(adb devices | Select-Object -Skip 1 | ForEach-Object { $_.Trim() } | Where-Object { $_ })
                    $emulatorLines = @($deviceLines | Where-Object { $_.StartsWith('emulator-') -and $_.EndsWith('device') })

                    if ($emulatorLines.Count -gt 0) {
                        $normalizedLine = $emulatorLines[0].Replace("`t", " ")
                        $emuSerial = $normalizedLine.Split(' ', [System.StringSplitOptions]::RemoveEmptyEntries)[0]
                        break
                    }

                    Start-Sleep -Seconds 5
                }

                if (-not $emuSerial) {
                    Write-Host "Saida atual do adb devices:"
                    adb devices
                    throw "Nenhum emulador Android online encontrado apos 2 minutos."
                }

                Write-Host "Emulador selecionado: $emuSerial"
                adb -s $emuSerial wait-for-device | Out-Null

                $packages = adb -s $emuSerial shell pm list packages
                $hasExpoGo = @($packages) -match 'host.exp.exponent'

                if (-not $hasExpoGo) {
                    Write-Host "Baixando Expo Go..."
                    $apkPath = Join-Path $PWD 'expo-go.apk'
                    Invoke-WebRequest -Uri 'https://d1ahtucjixef4r.cloudfront.net/Exponent-2.31.3.apk' -OutFile $apkPath -UseBasicParsing

                    Write-Host "Instalando Expo Go no emulador..."
                    adb -s $emuSerial install -r $apkPath | Out-Host
                }

                Write-Host "Expo Go instalado"
                '''

                powershell '''
                $ErrorActionPreference = 'Stop'

                if (-not (Get-Command java -ErrorAction SilentlyContinue)) {
                    throw "Java nao encontrado no agente Jenkins. O Maestro CLI requer Java 17+."
                }

                $maestroBin = $null
                $systemMaestro = Get-Command maestro -ErrorAction SilentlyContinue
                if ($systemMaestro) {
                    $maestroBin = $systemMaestro.Source
                    Write-Host "Usando Maestro global: $maestroBin"
                }
                else {
                    $cacheRoot = if ($env:JENKINS_HOME) { Join-Path $env:JENKINS_HOME 'cache/maestro-cli' } else { Join-Path $env:WORKSPACE '.tools/maestro-cache' }
                    $maestroDir = Join-Path $cacheRoot 'maestro'
                    $maestroZip = Join-Path $cacheRoot 'maestro.zip'

                    New-Item -ItemType Directory -Path $cacheRoot -Force | Out-Null

                    function Find-MaestroBin {
                        param([string]$Root)

                        $candidateNames = @('maestro.bat', 'maestro.cmd', 'maestro.exe', 'maestro')
                        $candidates = Get-ChildItem -Path $Root -Recurse -File -ErrorAction SilentlyContinue |
                            Where-Object { $candidateNames -contains $_.Name } |
                            Sort-Object FullName

                        if ($candidates.Count -eq 0) {
                            return $null
                        }

                        $preferred = $candidates | Where-Object { $_.Name -eq 'maestro.bat' } | Select-Object -First 1
                        if (-not $preferred) {
                            $preferred = $candidates | Select-Object -First 1
                        }

                        return $preferred.FullName
                    }

                    $maestroBin = Find-MaestroBin -Root $maestroDir

                    $cacheValid = $false
                    if ($maestroBin) {
                        try {
                            & $maestroBin --version | Out-Host
                            $cacheValid = $true
                            Write-Host "Reutilizando Maestro em cache: $maestroBin"
                        }
                        catch {
                            Write-Host "Cache do Maestro invalido; reinstalando..."
                        }
                    }

                    if (-not $cacheValid) {
                        Write-Host "Baixando Maestro CLI para cache..."
                        Invoke-WebRequest -Uri 'https://github.com/mobile-dev-inc/maestro/releases/latest/download/maestro.zip' -OutFile $maestroZip -UseBasicParsing

                        if (Test-Path $maestroDir) {
                            Remove-Item -Path $maestroDir -Recurse -Force
                        }

                        Expand-Archive -Path $maestroZip -DestinationPath $maestroDir -Force
                        $maestroBin = Find-MaestroBin -Root $maestroDir

                        if (-not $maestroBin) {
                            Write-Host "Conteudo extraido em $maestroDir (amostra):"
                            Get-ChildItem -Path $maestroDir -Recurse -File | Select-Object -First 20 FullName | Format-Table -AutoSize | Out-Host
                            throw "Nao foi possivel localizar o executavel do Maestro apos extrair maestro.zip."
                        }

                        Write-Host "Maestro local instalado em: $maestroBin"
                    }
                }

                $wrapperPath = Join-Path $env:WORKSPACE 'maestro-local.cmd'
                $wrapperContent = @(
                    '@echo off',
                    ('"{0}" %*' -f $maestroBin)
                )
                Set-Content -Path $wrapperPath -Value $wrapperContent -Encoding ASCII

                & $wrapperPath --version
                '''

                bat 'if not exist test-results mkdir test-results'
            }
        }

        stage('Flow - Login Admin') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/auth/login-admin.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/login-admin.xml'
                    }
                }
            }
        }

        stage('Flow - Login Cliente') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/auth/login-cliente.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/login-cliente.xml'
                    }
                }
            }
        }

        stage('Flow - Login Invalido') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/auth/login-credenciais-invalidas.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/login-invalido.xml'
                    }
                }
            }
        }

        stage('Flow - Register') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/auth/register.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/register.xml'
                    }
                }
            }
        }

        stage('Flow - Browse Products') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/customer/browse-products.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/browse-products.xml'
                    }
                }
            }
        }

        stage('Flow - Add To Cart') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/customer/add-to-cart.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/add-to-cart.xml'
                    }
                }
            }
        }

        stage('Flow - Remove From Cart') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/customer/remove-from-cart.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/remove-from-cart.xml'
                    }
                }
            }
        }

        stage('Flow - Checkout') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    timeout(time: 4, unit: 'MINUTES') {
                        bat '.\\maestro-local.cmd test .maestro/flows/customer/checkout.yaml -e EXPO_URL=%EXPO_URL% --format junit --output test-results/checkout.xml'
                    }
                }
            }
        }

        stage('Validate Maestro Results') {
            steps {
                powershell '''
                $ErrorActionPreference = 'Stop'

                $expectedFiles = @(
                    'test-results/login-admin.xml',
                    'test-results/login-cliente.xml',
                    'test-results/login-invalido.xml',
                    'test-results/register.xml',
                    'test-results/browse-products.xml',
                    'test-results/add-to-cart.xml',
                    'test-results/remove-from-cart.xml',
                    'test-results/checkout.xml'
                )

                $missingFiles = @($expectedFiles | Where-Object { -not (Test-Path $_) })
                if ($missingFiles.Count -gt 0) {
                    Write-Host 'Arquivos de resultado ausentes:'
                    $missingFiles | ForEach-Object { Write-Host " - $_" }
                    throw 'Nenhum ou alguns testes nao geraram JUnit XML. Falhando o stage para evitar falso positivo.'
                }

                $executedFiles = 0
                foreach ($file in $expectedFiles) {
                    [xml]$xmlDoc = Get-Content $file
                    $tests = 0

                    if ($xmlDoc.testsuite) {
                        $tests = [int]$xmlDoc.testsuite.tests
                    }
                    elseif ($xmlDoc.testsuites -and $xmlDoc.testsuites.testsuite) {
                        $tests = ($xmlDoc.testsuites.testsuite | Measure-Object -Property tests -Sum).Sum
                    }

                    if ($tests -gt 0) {
                        $executedFiles++
                    }
                }

                if ($executedFiles -eq 0) {
                    throw 'Nenhum teste foi realmente executado (todos os XML com tests=0).'
                }

                Write-Host "Validacao JUnit OK: $executedFiles arquivo(s) com testes executados."
                '''
            }
        }

    }

    post {

        always {

            script {
                def resultFiles = [
                    'test-results/login-admin.xml',
                    'test-results/login-cliente.xml',
                    'test-results/login-invalido.xml',
                    'test-results/register.xml',
                    'test-results/browse-products.xml',
                    'test-results/add-to-cart.xml',
                    'test-results/remove-from-cart.xml',
                    'test-results/checkout.xml'
                ]

                int totalTests = 0
                int totalFailures = 0
                int totalSkipped = 0
                int parsedFiles = 0

                resultFiles.each { file ->
                    if (!fileExists(file)) {
                        return
                    }

                    try {
                        def content = readFile(file)
                        parsedFiles++

                        totalTests += ((content =~ /(?i)<testcase\b/).count)
                        totalFailures += ((content =~ /(?i)<failure\b/).count)
                        totalSkipped += ((content =~ /(?i)<skipped\b/).count)
                    }
                    catch (e) {
                        echo "Aviso: nao foi possivel ler ${file}: ${e.message}"
                    }
                }

                if (parsedFiles == 0) {
                    echo 'Aviso: nenhum XML de resultado encontrado para montar o resumo.'
                }

                int totalPassed = totalTests - totalFailures - totalSkipped
                def summary = "Tests: ${totalTests} | Passed: ${totalPassed} | Failed: ${totalFailures} | Skipped: ${totalSkipped}"

                currentBuild.description = summary
                writeFile file: 'test-results/summary.txt', text: summary + "\n"

                echo summary
            }

            archiveArtifacts artifacts: 'test-results/summary.txt',
                             allowEmptyArchive: true

            junit allowEmptyResults: true,
                  testResults: 'test-results/*.xml'

            archiveArtifacts artifacts: '.maestro/debug*/**/*',
                             allowEmptyArchive: true

            archiveArtifacts artifacts: 'expo-server.log',
                             allowEmptyArchive: true

            bat '''
            @echo off
            for /f "tokens=1" %%i in ('adb devices ^| findstr "emulator"') do (
                adb -s %%i emu kill 2>nul
            )
            exit /b 0
            '''

            bat '''
            cd ecommerce-fullstack

            if errorlevel 1 (
                echo Pasta ecommerce-fullstack nao encontrada no workspace, ignorando cleanup do Docker.
                exit /b 0
            )

            docker compose down -v || echo Docker compose down retornou erro, seguindo cleanup.
            exit /b 0
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
