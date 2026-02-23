pipeline {
  agent any

  // Parámetros de ejecución
  parameters {
    string(name: 'MI_HOST', defaultValue: 'localhost', description: 'Host o IP donde está Micro Integrator', trim: true)
    string(name: 'MI_MGMT_PORT', defaultValue: '9164', description: 'Puerto Management API de MI', trim: true)
    booleanParam(name: 'MI_TLS_INSEGURO', defaultValue: true, description: 'Aceptar certificado TLS no confiable (dev)')
    booleanParam(name: 'COMPROBAR_HTTP', defaultValue: false, description: 'Comprobar endpoint runtime tras desplegar')
    string(name: 'MI_RUNTIME_PORT', defaultValue: '8290', description: 'Puerto runtime HTTP de MI', trim: true)
    string(name: 'HEALTH_PATH', defaultValue: '/patients/', description: 'Ruta a probar tras el despliegue', trim: true)

    string(name: 'APIM_HOST', defaultValue: 'apim.local', description: 'Host API Manager', trim: true)
    string(name: 'APIM_PORT', defaultValue: '9443', description: 'Puerto API Manager', trim: true)

    // Opcionales: si tu openapi no contiene name/version, pásalos aquí para publicar automáticamente
    string(name: 'API_NAME', defaultValue: '', description: 'Nombre de la API (opcional para publish automático)', trim: true)
    string(name: 'API_VERSION', defaultValue: '', description: 'Versión de la API (opcional para publish automático)', trim: true)
  }

  // Mapeo seguro de params a environment para usar en los scripts Windows (%VAR%)
  environment {
    MI_HOST       = "${params.MI_HOST ?: 'localhost'}"
    MI_MGMT_PORT  = "${params.MI_MGMT_PORT ?: '9164'}"
    // Convertir boolean -> texto ("true"/"false") para uso cómodo en batch
    MI_TLS_INSEGURO = "${params.MI_TLS_INSEGURO}"
    MI_RUNTIME_PORT = "${params.MI_RUNTIME_PORT ?: '8290'}"
    HEALTH_PATH     = "${params.HEALTH_PATH ?: '/patients/'}"

    APIM_HOST     = "${params.APIM_HOST ?: 'apim.local'}"
    APIM_PORT     = "${params.APIM_PORT ?: '9443'}"
    API_NAME      = "${params.API_NAME ?: ''}"
    API_VERSION   = "${params.API_VERSION ?: ''}"

    // WORKSPACE ya está disponible en el entorno de Jenkins
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    timeout(time: 60, unit: 'MINUTES')
    buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '30'))
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Checkout branch 'main' desde GitHub"
        git branch: 'main', url: 'https://github.com/youssefelouahabi2003/patient_repository.git'
      }
    }

    stage('Build (Maven)') {
      steps {
        echo "Ejecutando: mvn -B -DskipTests clean package"
        bat 'mvn -B -DskipTests clean package'
      }
    }

    stage('Verificar .car') {
      steps {
        echo "Comprobando que se generó al menos 1 .car en target\\"
        bat '''
          @echo off
          if exist target\\*.car (
            echo CARs encontrados:
            dir /B target\\*.car
          ) else (
            echo ERROR: No se generó ningún .car en target\\
            exit /b 1
          )
        '''
      }
    }

    stage('Desplegar en Micro Integrator (Windows)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'MI_ADMIN', usernameVariable: 'MI_USER', passwordVariable: 'MI_PASS')]) {
          bat '''
            @echo off
            setlocal enabledelayedexpansion

            REM -- asignar vars desde entorno de Jenkins --
            set "MI_HOST=%MI_HOST%"
            set "MI_MGMT_PORT=%MI_MGMT_PORT%"
            set "MI_USER=%MI_USER%"
            set "MI_PASS=%MI_PASS%"
            set "MI_TLS_INSEGURO=%MI_TLS_INSEGURO%"

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

            REM Obtener JWT (AccessToken)
            if "%MI_TLS_INSEGURO%"=="true" (
              curl -k -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json -w "\\nHTTP_STATUS=%{http_code}\\n"
            ) else (
              curl -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json -w "\\nHTTP_STATUS=%{http_code}\\n"
            )

            type login.json

            REM Extraer AccessToken con PowerShell (o con jq si disponible)
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

              if "%MI_TLS_INSEGURO%"=="true" (
                curl -k -f -sS -X POST "%APPS%" ^
                  -H "Authorization: Bearer %TOKEN%" ^
                  -H "Accept: application/json" ^
                  -F "file=@%%F" ^
                  -w "\\nHTTP_STATUS=%{http_code}\\n"
              ) else (
                curl -f -sS -X POST "%APPS%" ^
                  -H "Authorization: Bearer %TOKEN%" ^
                  -H "Accept: application/json" ^
                  -F "file=@%%F" ^
                  -w "\\nHTTP_STATUS=%{http_code}\\n"
              )

              if errorlevel 1 (
                echo ERROR: fallo subiendo %%~nxF
                exit /b 1
              )
            )

            echo Despliegue por API completado.
            endlocal
            exit /b 0
          '''
        }
      }
    }

    stage('Publicar en API Manager (Windows)') {
      when { expression { return true } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'APIM_ADMIN', usernameVariable: 'APIM_USER', passwordVariable: 'APIM_PASS')]) {
          bat '''
            @echo off
            setlocal enabledelayedexpansion

            REM Variables desde Jenkins environment
            set "APIM_HOST=%APIM_HOST%"
            set "APIM_PORT=%APIM_PORT%"
            set "APIM_USER=%APIM_USER%"
            set "APIM_PASS=%APIM_PASS%"
            set "OAS_FILE=%WORKSPACE%\\openapi.yaml"
            set "API_NAME=%API_NAME%"
            set "API_VERSION=%API_VERSION%"

            if not exist "%OAS_FILE%" (
              echo ERROR: no encuentro %OAS_FILE%
              exit /b 1
            )

            echo Comprobando apictl...
            where apictl >nul 2>nul
            if %ERRORLEVEL%==0 (
              echo apictl encontrado -> uso apictl para import/update
              apictl login ci -u "%APIM_USER%" -p "%APIM_PASS%" -k --host "https://%APIM_HOST%:%APIM_PORT%"
              if %ERRORLEVEL% neq 0 (
                echo ERROR: apictl login falló
                exit /b 1
              )

              echo Importando/actualizando API desde %OAS_FILE%...
              apictl import-api -f "%OAS_FILE%" -e ci --update
              if %ERRORLEVEL% neq 0 (
                echo ERROR: apictl import-api ha fallado
                exit /b 1
              )

              if not "%API_NAME%"=="" if not "%API_VERSION%"=="" (
                echo Publicando API %API_NAME% %API_VERSION% ...
                apictl change-status api -a Publish -n "%API_NAME%" -v "%API_VERSION%" -r "%APIM_USER%" -e ci
                if %ERRORLEVEL% neq 0 (
                  echo ERROR: apictl change-status ha fallado
                  exit /b 1
                )
                echo API publicada correctamente.
              ) else (
                echo Aviso: API_NAME/API_VERSION no proporcionados. Import realizado (puede quedar en CREATED).
              )

            ) else (
              echo apictl NO encontrado -> fallback REST import

              curl -k -s -o import_resp.json -w "HTTP_STATUS=%{http_code}" -u "%APIM_USER%:%APIM_PASS%" -F "file=@%OAS_FILE%" "https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v1/apis/import-openapi" > import_status.txt

              set /p HTTP_CODE=<import_status.txt
              echo Import REST HTTP status: %HTTP_CODE%

              if not "%HTTP_CODE%"=="200" if not "%HTTP_CODE%"=="201" (
                echo ERROR: fallo importando via REST (HTTP %HTTP_CODE%)
                echo Contenido import_resp.json:
                type import_resp.json || true
                exit /b 1
              )

              echo Import via REST completado. Nota: publish/lifecycle puede necesitar un cambio adicional via Product/Lifecycle APIs.
            )

            endlocal
          '''
        }
      }
    }

    stage('Comprobación HTTP (opcional)') {
      when { expression { return params.COMPROBAR_HTTP } }
      steps {
        echo "Comprobación del endpoint runtime en MI (hasta 1 min, polling cada 5s)"
        bat '''
          @echo off
          setlocal enabledelayedexpansion
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
          endlocal
        '''
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'target/*.car', fingerprint: true
      echo 'Pipeline finalizado con éxito.'
    }
    failure {
      echo 'Pipeline fallido. Revisa la consola para detalles.'
    }
    always {
      echo 'Post: limpieza/registro final.'
    }
  }
}