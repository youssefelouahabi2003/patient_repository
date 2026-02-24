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
        setlocal enabledelayedexpansion

        rem ====== APIM fijo a tu entorno ======
        set "APIM_HOST=localhost"
        set "APIM_PORT=9443"

        rem ====== Datos del API ======
        set "API_NAME=pepeprueba"
        set "API_VERSION=1.0.0"
        set "API_CONTEXT=/pepedoctor"

        rem ====== Backend (MI runtime) ======
        set "BACKEND_URL=http://localhost:8290"

        rem ====== Swagger/OpenAPI (ruta fija en tu repo) ======
        set "OAS_FILE=%WORKSPACE%\\src\\main\\wso2mi\\resources\\api-definitions\\HealthcareAPI1.yaml"

        if not exist "%OAS_FILE%" (
          echo ERROR: No existe el swagger/openapi en:
          echo   %OAS_FILE%
          exit /b 1
        )

        echo Usando definicion OpenAPI/Swagger:
        echo   %OAS_FILE%

        rem Cert TLS self-signed en local
        set "TLS=-k"

        rem ---- 1) DCR: crear OAuth app (clientId/clientSecret) ----
        set "DCR_URL=https://%APIM_HOST%:%APIM_PORT%/client-registration/v0.17/register"

        echo ------------------------------------------
        echo 1) DCR (registrar OAuth app)
        echo DCR_URL: %DCR_URL%
        echo ------------------------------------------

        set "DCR_PAYLOAD={\\"callbackUrl\\":\\"http://localhost\\",\\"clientName\\":\\"jenkins_publisher_api\\",\\"tokenScope\\":\\"Production\\",\\"owner\\":\\"%APIM_USER%\\",\\"grantType\\":\\"password refresh_token\\",\\"saasApp\\":true}"

        curl %TLS% -sS -u "%APIM_USER%:%APIM_PASS%" ^
          -H "Content-Type: application/json" ^
          -d "%DCR_PAYLOAD%" ^
          "%DCR_URL%" -o dcr.json

        powershell -NoProfile -Command "$d=ConvertFrom-Json (Get-Content dcr.json -Raw); $d.clientId" > client_id.txt
        powershell -NoProfile -Command "$d=ConvertFrom-Json (Get-Content dcr.json -Raw); $d.clientSecret" > client_secret.txt

        set /p CLIENT_ID=<client_id.txt
        set /p CLIENT_SECRET=<client_secret.txt

        if "%CLIENT_ID%"=="" (
          echo ERROR: DCR no devolvio clientId. Respuesta:
          type dcr.json
          exit /b 1
        )
        if "%CLIENT_SECRET%"=="" (
          echo ERROR: DCR no devolvio clientSecret. Respuesta:
          type dcr.json
          exit /b 1
        )

        rem ---- 2) Token OAuth2 (password grant) ----
        set "TOKEN_URL=https://%APIM_HOST%:%APIM_PORT%/oauth2/token"
        set "SCOPE=apim:api_view apim:api_create apim:api_manage"

        echo ------------------------------------------
        echo 2) Token OAuth2 (password grant)
        echo TOKEN_URL: %TOKEN_URL%
        echo ------------------------------------------

        curl %TLS% -sS -u "%CLIENT_ID%:%CLIENT_SECRET%" ^
          -H "Content-Type: application/x-www-form-urlencoded" ^
          -d "grant_type=password&username=%APIM_USER%&password=%APIM_PASS%&scope=%SCOPE%" ^
          "%TOKEN_URL%" -o apim_token.json

        powershell -NoProfile -Command "$t=ConvertFrom-Json (Get-Content apim_token.json -Raw); $t.access_token" > token.txt
        set /p APIM_TOKEN=<token.txt

        if "%APIM_TOKEN%"=="" (
          echo ERROR: No se pudo obtener access_token.
          type apim_token.json
          exit /b 1
        )

        echo ------------------------------------------
        echo TOKEN (MOSTRADO POR PETICION TUYA):
        echo %APIM_TOKEN%
        echo ------------------------------------------

        rem ---- 3) Buscar API existente (GET /apis = getAllAPIs) ----
        set "PUB_BASE=https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v4"

        rem IMPORTANTE: el %20 hay que ponerlo como %%20 en CMD
        set "LIST_URL=%PUB_BASE%/apis?query=name:!API_NAME!%%20version:!API_VERSION!"

        echo ------------------------------------------
        echo 3) Buscar API existente (GET /apis)
        echo LIST_URL: !LIST_URL!
        echo ------------------------------------------

        curl %TLS% -sS ^
          -H "Authorization: Bearer %APIM_TOKEN%" ^
          -H "Accept: application/json" ^
          "!LIST_URL!" -o apis.json

        rem Extraer API_ID sin FOR /F (evita el error "No se esperaba .")
        powershell -NoProfile -Command "$obj=ConvertFrom-Json (Get-Content apis.json -Raw); if($obj.count -gt 0){$obj.list[0].id}else{''}" > api_id.txt
        set /p API_ID=<api_id.txt

        if "%API_ID%"=="" (
          echo No existe el API en APIM: %API_NAME% %API_VERSION% (se creara).
          goto createApi
        ) else (
          echo API encontrado. ID=%API_ID% (se actualizara swagger).
          goto updateSwagger
        )

        :createApi
        rem ---- 4A) Crear API (POST /apis/import-openapi) ----
        set "IMPORT_URL=%PUB_BASE%/apis/import-openapi"

        set "ENDPOINTCFG={\\"endpoint_type\\":\\"http\\",\\"sandbox_endpoints\\":{\\"url\\":\\"%BACKEND_URL%\\"},\\"production_endpoints\\":{\\"url\\":\\"%BACKEND_URL%\\"}}"
        set "ADDITIONAL={\\"name\\":\\"%API_NAME%\\",\\"version\\":\\"%API_VERSION%\\",\\"context\\":\\"%API_CONTEXT%\\",\\"endpointConfig\\":\\"%ENDPOINTCFG%\\"}"

        echo ------------------------------------------
        echo 4A) Crear API (POST /apis/import-openapi)
        echo IMPORT_URL: %IMPORT_URL%
        echo BACKEND_URL: %BACKEND_URL%
        echo ------------------------------------------

        curl %TLS% -f -sS -X POST "%IMPORT_URL%" ^
          -H "Authorization: Bearer %APIM_TOKEN%" ^
          -H "Accept: application/json" ^
          -F "file=@%OAS_FILE%" ^
          -F "additionalProperties=%ADDITIONAL%" ^
          -o created_api.json

        if errorlevel 1 (
          echo ERROR: fallo creando API con import-openapi
          type created_api.json
          exit /b 1
        )

        powershell -NoProfile -Command "$c=ConvertFrom-Json (Get-Content created_api.json -Raw); $c.id" > api_id2.txt
        set /p API_ID=<api_id2.txt

        if "%API_ID%"=="" (
          echo ERROR: import-openapi no devolvio id de API
          type created_api.json
          exit /b 1
        )

        echo API creado. ID=%API_ID%
        goto done

        :updateSwagger
        rem ---- 4B) Actualizar Swagger (PUT /apis/{apiId}/swagger) ----
        set "SWAGGER_URL=%PUB_BASE%/apis/%API_ID%/swagger"

        echo ------------------------------------------
        echo 4B) Actualizar Swagger (PUT /apis/{apiId}/swagger)
        echo SWAGGER_URL: %SWAGGER_URL%
        echo ------------------------------------------

        curl %TLS% -f -sS -X PUT "%SWAGGER_URL%" ^
          -H "Authorization: Bearer %APIM_TOKEN%" ^
          -H "Accept: application/json" ^
          -F "file=@%OAS_FILE%" ^
          -o swagger_update.json

        if errorlevel 1 (
          echo ERROR: fallo actualizando swagger
          type swagger_update.json
          exit /b 1
        )

        echo Swagger actualizado OK para API_ID=%API_ID%

        :done
        echo ------------------------------------------
        echo APIM OK. API_ID=%API_ID%
        echo ------------------------------------------
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