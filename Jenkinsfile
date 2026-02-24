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

        rem ---------- Config ----------
        set "APIM_HOST=localhost"
        set "APIM_PORT=9443"

        set "API_NAME=pepeprueba"
        set "API_VERSION=1.0.0"
        set "API_CONTEXT=/pepedoctor"

        rem Ruta swagger en repo (fallback)
        set "REPO_OAS=%WORKSPACE%\\src\\main\\wso2mi\\resources\\api-definitions\\HealthcareAPI1.yaml"

        rem Carpeta temporal para extraer car
        set "CAR_EXTRACT=%WORKSPACE%\\car_extract"

        rem TLS insecure local
        set "TLS=-k"

        echo ---------------------------------------------------------
        echo 0) Buscar swagger: preferencia .car -> si no, fichero del repo
        echo ---------------------------------------------------------

        rem --- buscar .car con PowerShell y extraer swagger si existe ---
        powershell -NoProfile -Command ^
          "$car = Get-ChildItem -Path '%WORKSPACE%\\target' -Filter '*.car' -ErrorAction SilentlyContinue | Select-Object -First 1; if($car){ Write-Output $car.FullName }" > car_found.txt

        set "CAR_FILE="
        if exist car_found.txt (
          set /p CAR_FILE=<car_found.txt
        )

        if defined CAR_FILE (
          echo .car encontrado: %CAR_FILE%
          if exist "%CAR_EXTRACT%" rd /s /q "%CAR_EXTRACT%"
          powershell -NoProfile -Command "Expand-Archive -Force -Path '%CAR_FILE%' -DestinationPath '%CAR_EXTRACT%';"
          powershell -NoProfile -Command ^
            "$f = Get-ChildItem -Path '%CAR_EXTRACT%' -Recurse -Include '*.yaml','*.yml','*.json' -ErrorAction SilentlyContinue | Select-Object -First 1; if($f){ $f.FullName }" > swagger_path.txt
          if exist swagger_path.txt (
            set /p FOUND_SWAGGER=<swagger_path.txt
          )
        )

        if defined FOUND_SWAGGER (
          set "OAS_FILE=%FOUND_SWAGGER%"
          echo Swagger dentro de .car: %OAS_FILE%
        ) else (
          echo No se encontro swagger dentro del .car. Usando fichero del repo.
          set "OAS_FILE=%REPO_OAS%"
        )

        if not exist "%OAS_FILE%" (
          echo ERROR: No se encontro ningun swagger en .car ni en repo: %OAS_FILE%
          exit /b 1
        )

        echo Usando OpenAPI/Swagger: %OAS_FILE%
        echo ---------------------------------------------------------

        rem ----------------- DCR (crear OAuth client) -----------------
        set "DCR_URL=https://%APIM_HOST%:%APIM_PORT%/client-registration/v0.17/register"
        echo DCR_URL: %DCR_URL%

        rem crear dcr_payload.json sin BOM usando PowerShell
        powershell -NoProfile -Command ^
          "$o=@{callbackUrl='http://localhost';clientName='jenkins_publisher_api';tokenScope='Production';owner=$env:APIM_USER;grantType='password refresh_token';saasApp=$true};" ^
          "$s = $o | ConvertTo-Json -Compress; [System.IO.File]::WriteAllText('dcr_payload.json',$s,(New-Object System.Text.UTF8Encoding($false)))"

        curl %TLS% -sS -u "%APIM_USER%:%APIM_PASS%" -H "Content-Type: application/json" --data-binary @dcr_payload.json "%DCR_URL%" -o dcr.json
        if errorlevel 1 (
          echo ERROR: DCR curl fallo; ver dcr.json
          type dcr.json
          exit /b 1
        )

        powershell -NoProfile -Command "try{ (Get-Content dcr.json -Raw | ConvertFrom-Json).clientId } catch { Write-Output '' }" > client_id.txt
        powershell -NoProfile -Command "try{ (Get-Content dcr.json -Raw | ConvertFrom-Json).clientSecret } catch { Write-Output '' }" > client_secret.txt

        set /p CLIENT_ID=<client_id.txt
        set /p CLIENT_SECRET=<client_secret.txt

        if "%CLIENT_ID%"=="" (
          echo ERROR: DCR no devolvio clientId:
          type dcr.json
          exit /b 1
        )
        if "%CLIENT_SECRET%"=="" (
          echo ERROR: DCR no devolvio clientSecret:
          type dcr.json
          exit /b 1
        )

        echo DCR OK. clientId y secret obtenidos.

        rem ----------------- Obtener token (password grant) -----------------
        set "TOKEN_URL=https://%APIM_HOST%:%APIM_PORT%/oauth2/token"
        echo TOKEN_URL: %TOKEN_URL%

        rem usar --data-urlencode para evitar problemas con & en cmd
        curl %TLS% -sS -u "%CLIENT_ID%:%CLIENT_SECRET%" -H "Content-Type: application/x-www-form-urlencoded" ^
          --data-urlencode "grant_type=password" ^
          --data-urlencode "username=%APIM_USER%" ^
          --data-urlencode "password=%APIM_PASS%" ^
          --data-urlencode "scope=apim:api_view apim:api_create apim:api_manage" ^
          "%TOKEN_URL%" -o apim_token.json

        if errorlevel 1 (
          echo ERROR: token curl fallo; ver apim_token.json
          type apim_token.json
          exit /b 1
        )

        powershell -NoProfile -Command "try{ (Get-Content apim_token.json -Raw | ConvertFrom-Json).access_token } catch { Write-Output '' }" > token.txt
        set /p APIM_TOKEN=<token.txt

        if "%APIM_TOKEN%"=="" (
          echo ERROR: no access_token; ver apim_token.json
          type apim_token.json
          exit /b 1
        )

        echo Token obtenido (oculto).
        echo ---------------------------------------------------------

        rem ----------------- Crear API (import-openapi) -----------------
        set "PUB_BASE=https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v4"
        set "IMPORT_URL=%PUB_BASE%/apis/import-openapi"

        rem crear additional.json simple (sin BOM)
        powershell -NoProfile -Command ^
          "$json='{\"name\":\"%API_NAME%\",\"version\":\"%API_VERSION%\",\"context\":\"%API_CONTEXT%\"}'; [System.IO.File]::WriteAllText('additional.json',$json,(New-Object System.Text.UTF8Encoding($false)))"

        echo POST import-openapi -> %IMPORT_URL%
        curl %TLS% -sS -X POST "%IMPORT_URL%" -H "Authorization: Bearer %APIM_TOKEN%" -H "Accept: application/json" ^
          -F "file=@%OAS_FILE%" ^
          -F "additionalProperties=@additional.json" -o created_api.json

        if errorlevel 1 (
          echo ERROR: import-openapi fallo; ver created_api.json
          type created_api.json
          exit /b 1
        )

        powershell -NoProfile -Command "try{ (Get-Content created_api.json -Raw | ConvertFrom-Json).id } catch { Write-Output '' }" > api_id.txt
        set /p API_ID=<api_id.txt

        if "%API_ID%"=="" (
          echo ERROR: import-openapi no devolvio id; ver created_api.json
          type created_api.json
          exit /b 1
        )

        echo API creado. ID=%API_ID%

        rem ----------------- Actualizar endpoint (PUT /apis/{apiId}) -----------------
        rem crear update_body.json sin BOM con endpointConfig
        powershell -NoProfile -Command ^
          "$body = '{\"endpointConfig\": {\"endpoint_type\": \"http\", \"production_endpoints\": {\"url\": \"http://localhost:8290\"}, \"sandbox_endpoints\": {\"url\": \"http://localhost:8290\"}}}'; [System.IO.File]::WriteAllText('update_body.json',$body,(New-Object System.Text.UTF8Encoding($false)))"

        echo PUT update API endpoint for %API_ID%
        curl %TLS% -sS -X PUT "%PUB_BASE%/apis/%API_ID%" -H "Authorization: Bearer %APIM_TOKEN%" -H "Content-Type: application/json" -d @update_body.json -o update_api_result.json

        if errorlevel 1 (
          echo ERROR: fallo update endpoint; ver update_api_result.json
          type update_api_result.json
          exit /b 1
        )

        echo Endpoint actualizado. Revisa update_api_result.json si quieres detalles.

        echo ---------------------------------------------------------
        echo FIN STAGE APIM. API_ID=%API_ID%
        echo ---------------------------------------------------------
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