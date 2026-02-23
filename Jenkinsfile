pipeline {
  agent any

  parameters {
    string(name: 'MI_HOST', defaultValue: 'localhost', description: 'Host o IP donde está Micro Integrator', trim: true)
    string(name: 'MI_MGMT_PORT', defaultValue: '9164', description: 'Puerto Management API de MI', trim: true)
    booleanParam(name: 'MI_TLS_INSEGURO', defaultValue: true, description: 'Aceptar certificado TLS no confiable (dev)')
    booleanParam(name: 'COMPROBAR_HTTP', defaultValue: false, description: 'Comprobar endpoint runtime tras desplegar')
    string(name: 'MI_RUNTIME_PORT', defaultValue: '8290', description: 'Puerto runtime HTTP de MI', trim: true)
    string(name: 'HEALTH_PATH', defaultValue: '/patients/', description: 'Ruta a probar tras el despliegue', trim: true)

    string(name: 'APIM_HOST', defaultValue: 'localhost', description: 'Host API Manager', trim: true)
    string(name: 'APIM_PORT', defaultValue: '9443', description: 'Puerto API Manager', trim: true)

    string(name: 'API_NAME', defaultValue: '', description: 'Nombre de la API (opcional para publish automático)', trim: true)
    string(name: 'API_VERSION', defaultValue: '', description: 'Versión de la API (opcional para publish automático)', trim: true)
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
        // Versión robusta para Windows: evita paréntesis en echo dentro de un if-block
        bat """
          @echo off
          REM Buscamos archivos .car en target\
          set "FOUND=0"
          for %%F in (target\\*.car) do (
            set "FOUND=1"
            echo Found CAR: %%~nxF
          )
          if "%FOUND%"=="0" (
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

    stage('Publicar API en API Manager (Windows - REST, admin:admin)') {
      steps {
        bat """
          @echo off

          set "APIM_HOST=${params.APIM_HOST}"
          set "APIM_PORT=${params.APIM_PORT}"
          set "OAS_FILE=%WORKSPACE%\\openapi.yaml"

          if not exist "%OAS_FILE%" (
            echo ERROR: No se encuentra openapi.yaml
            exit /b 1
          )

          echo ==========================================
          echo 1) Importar OpenAPI en Publisher
          echo ==========================================

          curl -k -s -u "admin:admin" ^
            -F "file=@%OAS_FILE%" ^
            "https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v1/apis/import-openapi?preserveProvider=false" ^
            -o import_response.json

          if errorlevel 1 (
            echo ERROR: Fallo al importar la API
            type import_response.json
            exit /b 1
          )

          echo Respuesta import:
          type import_response.json

          echo ==========================================
          echo 2) Extraer API ID
          echo ==========================================

          for /f "usebackq delims=" %%A in (`powershell -NoProfile -Command ^
            "(Get-Content import_response.json -Raw | ConvertFrom-Json).id"`) do set "API_ID=%%A"

          if "%API_ID%"=="" (
            echo ERROR: No se pudo obtener API_ID
            exit /b 1
          )

          echo API_ID = %API_ID%

          echo ==========================================
          echo 3) Publicar API (Change Lifecycle)
          echo ==========================================

          powershell -NoProfile -Command ^
            "$body = @{ action='Publish'; apiId='%API_ID%'; lifecycleChecklist=@() } | ConvertTo-Json -Compress; ^
             Invoke-RestMethod -Method Post ^
               -Uri 'https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v1/apis/change-lifecycle' ^
               -Body $body ^
               -ContentType 'application/json' ^
               -Credential (New-Object System.Management.Automation.PSCredential('admin',(ConvertTo-SecureString 'admin' -AsPlainText -Force))) ^
               -SkipCertificateCheck"

          if errorlevel 1 (
            echo ERROR: Fallo al publicar la API
            exit /b 1
          )

          echo ==========================================
          echo API IMPORTADA Y PUBLICADA CORRECTAMENTE
          echo ==========================================
        """
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