pipeline {
  agent any

  parameters {
    string(name: 'MI_HOST', defaultValue: 'localhost', description: 'Host o IP donde está Micro Integrator', trim: true)
    string(name: 'MI_MGMT_PORT', defaultValue: '9164', description: 'Puerto Management API de MI', trim: true)
    booleanParam(name: 'MI_TLS_INSEGURO', defaultValue: true, description: 'Aceptar certificado TLS no confiable (dev)')
    booleanParam(name: 'COMPROBAR_HTTP', defaultValue: false, description: 'Comprobar endpoint runtime tras desplegar')
    string(name: 'MI_RUNTIME_PORT', defaultValue: '8290', description: 'Puerto runtime HTTP de MI', trim: true)
    string(name: 'HEALTH_PATH', defaultValue: '/patients/', description: 'Ruta a probar tras el despliegue', trim: true)


  }

  options {
    timestamps()
    disableConcurrentBuilds()
    timeout(time: 45, unit: 'MINUTES')
    buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '30'))
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/youssefelouahabi2003/patient_repository.git'
      }
    }

    stage('Build (Maven)') {
      steps {
        bat 'mvn -B -DskipTests clean package'
      }
    }

    stage('Verificar .car') {
      steps {
        bat """
          @echo off
          if exist target\\*.car (
            echo CARs encontrados:
            dir /B target\\*.car
          ) else (
            echo ERROR: No se generó ningún .car en target\\
            exit /b 1
          )
        """
      }
    }

    stage('Desplegar en Micro Integrator (Windows)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'MI_ADMIN', usernameVariable: 'MI_USER', passwordVariable: 'MI_PASS')]) {
          bat """
            @echo off
            REM -- asignar vars --
            set "MI_HOST=${params.MI_HOST}"
            set "MI_MGMT_PORT=${params.MI_MGMT_PORT}"
            set "MI_USER=%MI_USER%"
            set "MI_PASS=%MI_PASS%"

            if "%MI_HOST%"=="" (
              echo ERROR: MI_HOST vacio
              exit /b 1
            )
            if "%MI_MGMT_PORT%"=="" (
              echo ERROR: MI_MGMT_PORT vacio
              exit /b 1
            )

            set "BASE=https://%MI_HOST%:%MI_MGMT_PORT%/management"
            set "LOGIN=%BASE%/login"
            set "APPS=%BASE%/applications"

            echo ------------------------------------------
            echo 1) Login para obtener JWT
            echo LOGIN: %LOGIN%
            echo ------------------------------------------

            rem Obtener JWT (AccessToken)
            if "${params.MI_TLS_INSEGURO}"=="true" (
              curl -k -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json -w "\\nHTTP_STATUS=%{http_code}\\n"
            ) else (
              curl -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json -w "\\nHTTP_STATUS=%{http_code}\\n"
            )

            type login.json

            rem Extraer AccessToken con PowerShell
            for /f "usebackq delims=" %%T in (`powershell -NoProfile -Command "(Get-Content login.json -Raw | ConvertFrom-Json).AccessToken"`) do set "TOKEN=%%T"

            if "%TOKEN%"=="" (
              echo ERROR: No se pudo obtener token. Revisa usuario/pass o config de MI.
              exit /b 1
            )

            echo Token obtenido (oculto).
            echo ------------------------------------------
            echo 2) Subir .car usando Bearer token
            echo APPS: %APPS%
            echo ------------------------------------------

            for %%F in ("%WORKSPACE%\\target\\*.car") do (
              echo Subiendo: %%~nxF

              if "${params.MI_TLS_INSEGURO}"=="true" (
                curl -k -f -sS -X POST "%APPS%" ^
                  -H "Authorization: Bearer %TOKEN%" ^
                  -H "Accept: application/json" ^
                  -F "file=@%%F" ^
                  -w "\\nHTTP_STATUS=%%{http_code}\\n"
              ) else (
                curl -f -sS -X POST "%APPS%" ^
                  -H "Authorization: Bearer %TOKEN%" ^
                  -H "Accept: application/json" ^
                  -F "file=@%%F" ^
                  -w "\\nHTTP_STATUS=%%{http_code}\\n"
              )

              if errorlevel 1 (
                echo ERROR: fallo subiendo %%~nxF
                exit /b 1
              )
            )

            echo Despliegue por API completado.
            exit /b 0
          """
        }
      }
    }

    stage('Publicar/Actualizar API en WSO2 API Manager (Publisher v4 - APIM 4.3 LOCAL)') {
  steps {
    withCredentials([
      usernamePassword(credentialsId: 'APIM_ADMIN', usernameVariable: 'APIM_USER', passwordVariable: 'APIM_PASS')
    ]) {
      bat '''
        @echo off
        setlocal

        rem ====== APIM fijo a tu entorno ======
        set "APIM_HOST=localhost"
        set "APIM_PORT=9443"

        rem ====== Datos del API ======
        set "API_NAME=pepeprueba"
        set "API_VERSION=1.0.0"
        set "API_CONTEXT=/pepedoctor"

        rem ====== Backend (MI runtime) ======
        set "BACKEND_URL=http://localhost:8290"

        rem ====== Swagger/OpenAPI (ruta fija) ======
        set "OAS_FILE=%WORKSPACE%\\src\\main\\wso2mi\\resources\\api-definitions\\HealthcareAPI1.yaml"
        if not exist "%OAS_FILE%" (
          echo ERROR: No existe el swagger/openapi en:
          echo   %OAS_FILE%
          exit /b 1
        )

        echo Usando definicion OpenAPI/Swagger:
        echo   %OAS_FILE%

        rem ---- Ejecutar PowerShell seguro (escapando & con ^& para CMD) ----
        powershell -NoProfile -ExecutionPolicy Bypass -Command ^
          "$ErrorActionPreference='Stop';" ^
          "[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12;" ^
          "[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true};" ^
          "$apimHost=$env:APIM_HOST; $apimPort=$env:APIM_PORT;" ^
          "$user=$env:APIM_USER; $pass=$env:APIM_PASS;" ^
          "$apiName=$env:API_NAME; $apiVersion=$env:API_VERSION; $apiContext=$env:API_CONTEXT;" ^
          "$backend=$env:BACKEND_URL;" ^
          "$oasFile=$env:OAS_FILE;" ^
          "Write-Host '------------------------------------------';" ^
          "Write-Host '1) DCR (registrar OAuth app)';" ^
          "$dcrUrl = \"https://$apimHost`:$apimPort/client-registration/v0.17/register\";" ^
          "Write-Host ('DCR_URL: ' + $dcrUrl);" ^
          "$basic = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(\"$user`:$pass\"));" ^
          "$dcrPayload = @{callbackUrl='http://localhost'; clientName='jenkins_publisher_api'; tokenScope='Production'; owner=$user; grantType='password refresh_token'; saasApp=$true} | ConvertTo-Json -Compress;" ^
          "$dcrResp = Invoke-RestMethod -Method Post -Uri $dcrUrl -Headers @{Authorization=\"Basic $basic\"; 'Content-Type'='application/json'} -Body $dcrPayload;" ^
          "$clientId = $dcrResp.clientId; $clientSecret = $dcrResp.clientSecret;" ^
          "if(-not $clientId -or -not $clientSecret){ throw ('DCR sin clientId/clientSecret. Respuesta: ' + ($dcrResp | ConvertTo-Json -Compress)) }" ^
          "Write-Host '------------------------------------------';" ^
          "Write-Host '2) Token OAuth2 (password grant)';" ^
          "$tokenUrl = \"https://$apimHost`:$apimPort/oauth2/token\";" ^
          "Write-Host ('TOKEN_URL: ' + $tokenUrl);" ^
          "$scope = 'apim:api_view apim:api_create apim:api_manage';" ^
          "$basicApp = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(\"$clientId`:$clientSecret\"));" ^
          # --------- IMPORTANT: escape & as ^& so CMD doesn't split the line ----------
          "$body = \"grant_type=password^&username=$([uri]::EscapeDataString($user))^&password=$([uri]::EscapeDataString($pass))^&scope=$([uri]::EscapeDataString($scope))\";" ^
          # -------------------------------------------------------------------------
          "$tokResp = Invoke-RestMethod -Method Post -Uri $tokenUrl -Headers @{Authorization=\"Basic $basicApp\"; 'Content-Type'='application/x-www-form-urlencoded'} -Body $body;" ^
          "$token = $tokResp.access_token;" ^
          "if(-not $token){ throw ('No access_token. Respuesta: ' + ($tokResp | ConvertTo-Json -Compress)) }" ^
          "Write-Host '------------------------------------------';" ^
          "Write-Host 'TOKEN (MOSTRADO POR PETICION TUYA):';" ^
          "Write-Host $token;" ^
          "Write-Host '------------------------------------------';" ^
          "Write-Host '3) Buscar API existente (GET /apis)';" ^
          "$pubBase = \"https://$apimHost`:$apimPort/api/am/publisher/v4\";" ^
          "$q = \"name:$apiName version:$apiVersion\";" ^
          "$listUrl = \"$pubBase/apis?query=$([uri]::EscapeDataString($q))\";" ^
          "Write-Host ('LIST_URL: ' + $listUrl);" ^
          "$apis = Invoke-RestMethod -Method Get -Uri $listUrl -Headers @{Authorization=\"Bearer $token\"; Accept='application/json'} -ErrorAction Stop;" ^
          "$apiId = '';" ^
          "if($apis -and $apis.count -gt 0 -and $apis.list -and $apis.list.Count -gt 0){ $apiId = $apis.list[0].id }" ^
          "if([string]::IsNullOrWhiteSpace($apiId)){" ^
          "  Write-Host ('No existe el API en APIM: ' + $apiName + ' ' + $apiVersion + ' (se creara).');" ^
          "  $importUrl = \"$pubBase/apis/import-openapi\";" ^
          "  $endpointCfg = @{ endpoint_type='http'; sandbox_endpoints=@{url=$backend}; production_endpoints=@{url=$backend} } | ConvertTo-Json -Compress;" ^
          "  $additional = @{ name=$apiName; version=$apiVersion; context=$apiContext; endpointConfig=$endpointCfg } | ConvertTo-Json -Compress;" ^
          "  Add-Type -AssemblyName System.Net.Http;" ^
          "  $handler = New-Object System.Net.Http.HttpClientHandler;" ^
          "  $handler.ServerCertificateCustomValidationCallback = { $true };" ^
          "  $client = New-Object System.Net.Http.HttpClient($handler);" ^
          "  $client.DefaultRequestHeaders.Authorization = New-Object System.Net.Http.Headers.AuthenticationHeaderValue('Bearer', $token);" ^
          "  $content = New-Object System.Net.Http.MultipartFormDataContent;" ^
          "  $fileBytes = [IO.File]::ReadAllBytes($oasFile);" ^
          "  $fileContent = New-Object System.Net.Http.ByteArrayContent($fileBytes);" ^
          "  $fileContent.Headers.ContentType = [System.Net.Http.Headers.MediaTypeHeaderValue]::Parse('application/octet-stream');" ^
          "  $content.Add($fileContent, 'file', [IO.Path]::GetFileName($oasFile));" ^
          "  $content.Add((New-Object System.Net.Http.StringContent($additional)), 'additionalProperties');" ^
          "  $resp = $client.PostAsync($importUrl, $content).Result;" ^
          "  $respBody = $resp.Content.ReadAsStringAsync().Result;" ^
          "  if(-not $resp.IsSuccessStatusCode){ throw ('import-openapi fallo: ' + $resp.StatusCode + ' ' + $respBody) }" ^
          "  $created = $respBody | ConvertFrom-Json;" ^
          "  $apiId = $created.id;" ^
          "  if(-not $apiId){ throw ('import-openapi OK pero sin id: ' + $respBody) }" ^
          "  Write-Host ('API creado. ID=' + $apiId);" ^
          "} else {" ^
          "  Write-Host ('API encontrado. ID=' + $apiId + ' (se actualizara swagger).');" ^
          "  $swaggerUrl = \"$pubBase/apis/$apiId/swagger\";" ^
          "  Add-Type -AssemblyName System.Net.Http;" ^
          "  $handler = New-Object System.Net.Http.HttpClientHandler;" ^
          "  $handler.ServerCertificateCustomValidationCallback = { $true };" ^
          "  $client = New-Object System.Net.Http.HttpClient($handler);" ^
          "  $client.DefaultRequestHeaders.Authorization = New-Object System.Net.Http.Headers.AuthenticationHeaderValue('Bearer', $token);" ^
          "  $content = New-Object System.Net.Http.MultipartFormDataContent;" ^
          "  $fileBytes = [IO.File]::ReadAllBytes($oasFile);" ^
          "  $fileContent = New-Object System.Net.Http.ByteArrayContent($fileBytes);" ^
          "  $fileContent.Headers.ContentType = [System.Net.Http.Headers.MediaTypeHeaderValue]::Parse('application/octet-stream');" ^
          "  $content.Add($fileContent, 'file', [IO.Path]::GetFileName($oasFile));" ^
          "  $resp = $client.PutAsync($swaggerUrl, $content).Result;" ^
          "  $respBody = $resp.Content.ReadAsStringAsync().Result;" ^
          "  if(-not $resp.IsSuccessStatusCode){ throw ('update swagger fallo: ' + $resp.StatusCode + ' ' + $respBody) }" ^
          "  Write-Host ('Swagger actualizado OK para API_ID=' + $apiId);" ^
          "}" ^
          "Write-Host '------------------------------------------';" ^
          "Write-Host ('APIM OK. API_ID=' + $apiId);" ^
          "Write-Host '------------------------------------------';"

        if errorlevel 1 (
          echo ERROR: Stage APIM fallo. Revisa logs arriba (PowerShell).
          exit /b 1
        )

        exit /b 0
      '''
    }
  }
}

  

    stage('Comprobación HTTP (opcional)') {
      when { expression { return params.COMPROBAR_HTTP } }
      steps {
        bat """
          @echo off
          echo Esperando 10s a que MI procese el .car...
          timeout /t 10 /nobreak >nul

          set "URL=http://%MI_HOST%:%MI_RUNTIME_PORT%%HEALTH_PATH%"
          set ATTEMPTS=12
          set /a I=1
          :loop
          powershell -NoProfile -Command "(Invoke-WebRequest -UseBasicParsing -Uri '%URL%' -TimeoutSec 8).StatusCode" > status.txt 2>nul || echo ERROR > status.txt
          set /p CODE=<status.txt
          if "%CODE%"=="ERROR" (
            echo Intento %I%/%ATTEMPTS%: aún no responde %URL%. Esperamos 5s...
            timeout /t 5 /nobreak >nul
            set /a I+=1
            if %I% leq %ATTEMPTS% goto loop
            echo El endpoint no respondió: %URL%
            exit /b 1
          ) else (
            echo OK: respondió %URL% -> %CODE%
          )
        """
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'target/*.car', fingerprint: true
      echo 'Pipeline finalizado con éxito.'
    }
    failure {
      echo 'Pipeline fallido. Revisa la consola.'
    }
  }
}