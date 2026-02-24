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

stage('Publicar API desde Swagger (.car or repo)') {
  steps {
    withCredentials([
      usernamePassword(credentialsId: 'APIM_ADMIN', usernameVariable: 'APIM_USER', passwordVariable: 'APIM_PASS')
    ]) {
      bat '''
        @echo off
        setlocal

        set "APIM_HOST=localhost"
        set "APIM_PORT=9443"

        set "API_NAME=pepeprueba"
        set "API_VERSION=1.0.0"
        set "API_CONTEXT=/pepedoctor"

        set "REPO_OAS=%WORKSPACE%\\src\\main\\wso2mi\\resources\\api-definitions\\HealthcareAPI1.yaml"
        set "CAR_EXTRACT=%WORKSPACE%\\car_extract"
        set "TLS=-k"

        echo ---------------------------------------------------------
        echo 0) Buscar swagger: preferencia .car -> si no, fichero del repo
        echo ---------------------------------------------------------

        powershell -NoProfile -Command ^
          "$car = Get-ChildItem -Path '%WORKSPACE%\\target' -Filter '*.car' -ErrorAction SilentlyContinue | Select-Object -First 1; if($car){ $car.FullName }" > car_found.txt

        set "CAR_FILE="
        if exist car_found.txt set /p CAR_FILE=<car_found.txt

        set "FOUND_SWAGGER="
        if defined CAR_FILE (
          echo .car encontrado: %CAR_FILE%
          if exist "%CAR_EXTRACT%" rd /s /q "%CAR_EXTRACT%"
          powershell -NoProfile -Command "Expand-Archive -Force -Path '%CAR_FILE%' -DestinationPath '%CAR_EXTRACT%';"
          powershell -NoProfile -Command ^
            "$f = Get-ChildItem -Path '%CAR_EXTRACT%' -Recurse -Include '*.yaml','*.yml','*.json' -ErrorAction SilentlyContinue | Select-Object -First 1; if($f){ $f.FullName }" > swagger_path.txt
          if exist swagger_path.txt set /p FOUND_SWAGGER=<swagger_path.txt
        )

        if defined FOUND_SWAGGER (
          set "OAS_FILE=%FOUND_SWAGGER%"
          echo Swagger dentro de .car: %OAS_FILE%
        ) else (
          set "OAS_FILE=%REPO_OAS%"
          echo Usando swagger del repo: %OAS_FILE%
        )

        if not exist "%OAS_FILE%" (
          echo ERROR: No existe swagger: %OAS_FILE%
          exit /b 1
        )

        rem ----------------- DCR -----------------
        set "DCR_URL=https://%APIM_HOST%:%APIM_PORT%/client-registration/v0.17/register"
        powershell -NoProfile -Command ^
          "$o=@{callbackUrl='http://localhost';clientName='jenkins_publisher_api';tokenScope='Production';owner=$env:APIM_USER;grantType='password refresh_token';saasApp=$true};" ^
          "$s = $o | ConvertTo-Json -Compress; [System.IO.File]::WriteAllText('dcr_payload.json',$s,(New-Object System.Text.UTF8Encoding($false)))"

        curl %TLS% -sS -u "%APIM_USER%:%APIM_PASS%" -H "Content-Type: application/json" --data-binary @dcr_payload.json "%DCR_URL%" -o dcr.json || ( type dcr.json & exit /b 1 )

        powershell -NoProfile -Command "(Get-Content dcr.json -Raw | ConvertFrom-Json).clientId" > client_id.txt
        powershell -NoProfile -Command "(Get-Content dcr.json -Raw | ConvertFrom-Json).clientSecret" > client_secret.txt
        set /p CLIENT_ID=<client_id.txt
        set /p CLIENT_SECRET=<client_secret.txt

        rem ----------------- TOKEN -----------------
        set "TOKEN_URL=https://%APIM_HOST%:%APIM_PORT%/oauth2/token"
        curl %TLS% -sS -u "%CLIENT_ID%:%CLIENT_SECRET%" -H "Content-Type: application/x-www-form-urlencoded" ^
          --data-urlencode "grant_type=password" ^
          --data-urlencode "username=%APIM_USER%" ^
          --data-urlencode "password=%APIM_PASS%" ^
          --data-urlencode "scope=apim:api_view apim:api_create apim:api_manage" ^
          "%TOKEN_URL%" -o apim_token.json || ( type apim_token.json & exit /b 1 )

        powershell -NoProfile -Command "(Get-Content apim_token.json -Raw | ConvertFrom-Json).access_token" > token.txt
        set /p APIM_TOKEN=<token.txt

        echo Token obtenido (oculto).
        echo ---------------------------------------------------------

        rem ----------------- IMPORT OPENAPI -----------------
        set "PUB_BASE=https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v4"
        set "IMPORT_URL=%PUB_BASE%/apis/import-openapi"

        rem ✅ additional.json SIN BOM (robusto)
        powershell -NoProfile -Command ^
          "$json = '{\"name\":\"%API_NAME%\",\"version\":\"%API_VERSION%\",\"context\":\"%API_CONTEXT%\"}';" ^
          "[System.IO.File]::WriteAllText('additional.json', $json, (New-Object System.Text.UTF8Encoding($false)))"

        echo --- additional.json (debug) ---
        type additional.json

        echo POST import-openapi -> %IMPORT_URL%
        curl %TLS% -sS -X POST "%IMPORT_URL%" -H "Authorization: Bearer %APIM_TOKEN%" -H "Accept: application/json" ^
          -F "file=@%OAS_FILE%" ^
          -F "additionalProperties=@additional.json" -o created_api.json

        echo --- created_api.json (debug) ---
        type created_api.json

        powershell -NoProfile -Command "try{ (Get-Content created_api.json -Raw | ConvertFrom-Json).id } catch { '' }" > api_id.txt
        set /p API_ID=<api_id.txt

        if "%API_ID%"=="" (
          echo ERROR: import-openapi no devolvio id; ver created_api.json
          exit /b 1
        )

        echo API creado. ID=%API_ID%
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