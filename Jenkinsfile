pipeline {
  agent any

  parameters {
    string(name: 'MI_HOST', defaultValue: 'localhost', description: 'Host o IP donde está Micro Integrator', trim: true)
    string(name: 'MI_MGMT_PORT', defaultValue: '9164', description: 'Puerto Management API de MI', trim: true)
    booleanParam(name: 'MI_TLS_INSEGURO', defaultValue: true, description: 'Aceptar certificado TLS no confiable (dev)')
    booleanParam(name: 'COMPROBAR_HTTP', defaultValue: false, description: 'Comprobar endpoint runtime tras desplegar')
    string(name: 'MI_RUNTIME_PORT', defaultValue: '8290', description: 'Puerto runtime HTTP de MI', trim: true)
    string(name: 'HEALTH_PATH', defaultValue: '/patients/', description: 'Ruta a probar tras el despliegue', trim: true)

    string(name: 'APIM_HOST', defaultValue: 'apim.local', description: 'Host API Manager', trim: true)
    string(name: 'APIM_PORT', defaultValue: '9443', description: 'Puerto API Manager', trim: true)

    string(name: 'API_NAME', defaultValue: '', description: 'Nombre de la API (opcional para publish automático)', trim: true)
    string(name: 'API_VERSION', defaultValue: '', description: 'Versión de la API (opcional para publish automático)', trim: true)
  }

  environment {
    MI_HOST         = "${params.MI_HOST ?: 'localhost'}"
    MI_MGMT_PORT    = "${params.MI_MGMT_PORT ?: '9164'}"
    MI_TLS_INSEGURO = "${params.MI_TLS_INSEGURO}"
    MI_RUNTIME_PORT = "${params.MI_RUNTIME_PORT ?: '8290'}"
    HEALTH_PATH     = "${params.HEALTH_PATH ?: '/patients/'}"

    APIM_HOST       = "${params.APIM_HOST ?: 'apim.local'}"
    APIM_PORT       = "${params.APIM_PORT ?: '9443'}"
    API_NAME        = "${params.API_NAME ?: ''}"
    API_VERSION     = "${params.API_VERSION ?: ''}"
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

    stage('Diagnostic Mi Connectivity') {
      steps {
        echo "Verificaciones de conectividad y cabeceras para el endpoint de Management (Windows-safe)"
        bat '''
          @echo off
          setlocal

          set "MI_HOST=%MI_HOST%"
          set "MI_MGMT_PORT=%MI_MGMT_PORT%"
          set "LOGIN_URL=https://%MI_HOST%:%MI_MGMT_PORT%/management/login"

          echo -------------------------------------------------------
          echo 0) Resolución DNS y reachability
          echo -------------------------------------------------------
          nslookup %MI_HOST% > dns_lookup.txt 2>&1

          echo ==== dns_lookup.txt ====
          if exist dns_lookup.txt (
            type dns_lookup.txt
          ) else (
            echo dns_lookup.txt no creado
          )

          echo -------------------------------------------------------
          echo 1) Test-NetConnection (puerto TCP) desde PowerShell
          echo -------------------------------------------------------
          powershell -NoProfile -Command "try { Test-NetConnection -ComputerName '%MI_HOST%' -Port %MI_MGMT_PORT% | Out-File -Encoding ascii testnetconn.txt } catch { 'PS_FAILED' | Out-File -Encoding ascii testnetconn.txt }"

          echo ==== testnetconn.txt ====
          if exist testnetconn.txt (
            type testnetconn.txt
          ) else (
            echo testnetconn.txt no creado
          )

          echo -------------------------------------------------------
          echo 2) HEAD request (curl -I) y guardar headers (usa NUL en Windows)
          echo -------------------------------------------------------
          if "%MI_TLS_INSEGURO%"=="true" (
            curl -k -sS -D login_headers.txt -I "%LOGIN_URL%" -o NUL || echo "curl HEAD fallo" > login_headers.txt
          ) else (
            curl -sS -D login_headers.txt -I "%LOGIN_URL%" -o NUL || echo "curl HEAD fallo" > login_headers.txt
          )

          echo ==== login_headers.txt ====
          if exist login_headers.txt (
            type login_headers.txt
          ) else (
            echo login_headers.txt no creado
          )

          endlocal
        '''
      }
    }

    stage('Desplegar en Micro Integrator (Windows)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'MI_ADMIN', usernameVariable: 'MI_USER', passwordVariable: 'MI_PASS')]) {
          bat '''
            @echo off
            setlocal enabledelayedexpansion

            REM variables desde Jenkins
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

            echo -------------------------------------------------------
            echo 1) Intentando login (Basic Auth) para obtener JWT
            echo LOGIN: %LOGIN%
            echo -------------------------------------------------------

            REM Limpiar ficheros previos
            if exist login.json del /q login.json
            if exist login_status.txt del /q login_status.txt
            if exist login_debug.txt del /q login_debug.txt
            if exist login_headers.txt del /q login_headers.txt
            if exist token.txt del /q token.txt

            REM ---- intento 1: Basic Auth (verbose) ----
            echo [DEBUG] Intento BasicAuth (curl -v). stdout->login.json stderr->login_debug.txt, HTTP->login_status.txt
            if "%MI_TLS_INSEGURO%"=="true" (
              curl -k -v -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json --write-out "%%{http_code}" > login_status.txt 2>login_debug.txt
            ) else (
              curl -v -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json --write-out "%%{http_code}" > login_status.txt 2>login_debug.txt
            )

            if exist login_status.txt (
              set /p HTTP_CODE=<login_status.txt
            ) else (
              set "HTTP_CODE="
            )
            echo Login HTTP status: %HTTP_CODE%

            echo ==== login_debug.txt (stderr curl verbose) ====
            if exist login_debug.txt (
              type login_debug.txt
            ) else (
              echo login_debug.txt no creado
            )

            REM Guardar headers también para inspección (HEAD request) - NUL en Windows
            if "%MI_TLS_INSEGURO%"=="true" (
              curl -k -sS -D login_headers.txt -I "%LOGIN%" -o NUL || echo "HEAD_FAIL" > login_headers.txt
            ) else (
              curl -sS -D login_headers.txt -I "%LOGIN%" -o NUL || echo "HEAD_FAIL" > login_headers.txt
            )
            echo ==== login_headers.txt ====
            if exist login_headers.txt (
              type login_headers.txt
            ) else (
              echo login_headers.txt no creado
            )

            REM Si 200 -> parse token, si 401 -> intentar POST JSON
            if "%HTTP_CODE%"=="200" goto :parse_token

            echo NOT 200. Intentaremos POST JSON (si BasicAuth no funciona). (HTTP %HTTP_CODE%)

            REM ---- intento 2: POST JSON (verbose) ----
            echo [DEBUG] Intento POST JSON (curl -v)
            if "%MI_TLS_INSEGURO%"=="true" (
              curl -k -v -sS -X POST -H "Content-Type: application/json" -d "{\"username\":\"%MI_USER%\",\"password\":\"%MI_PASS%\"}" "%LOGIN%" -o login.json --write-out "%%{http_code}" > login_status.txt 2>login_debug.txt
            ) else (
              curl -v -sS -X POST -H "Content-Type: application/json" -d "{\"username\":\"%MI_USER%\",\"password\":\"%MI_PASS%\"}" "%LOGIN%" -o login.json --write-out "%%{http_code}" > login_status.txt 2>login_debug.txt
            )

            if exist login_status.txt (
              set /p HTTP_CODE=<login_status.txt
            ) else (
              set "HTTP_CODE="
            )
            echo Login POST JSON HTTP status: %HTTP_CODE%

            echo ==== login_debug.txt (stderr curl verbose) ====
            if exist login_debug.txt (
              type login_debug.txt
            ) else (
              echo login_debug.txt no creado
            )

            if not "%HTTP_CODE%"=="200" (
              echo ERROR: login a MI falló tras ambos intentos (HTTP %HTTP_CODE%).
              echo ==== login.json (respuesta) ====
              if exist login.json (
                type login.json
              ) else (
                echo login.json no creado
              )

              REM chequeo rápido si la respuesta es HTML (form/login page)
              if exist login.json (
                findstr /i "<html" login.json > nul 2>nul && echo Nota: la respuesta parece ser HTML (posible página de login) || echo Nota: la respuesta no parece HTML.
              )

              echo -------------------------------------------------------
              echo CHECKLIST SUGERIDO:
              echo  - Credenciales MI_ADMIN correctas en Jenkins?
              echo  - Endpoint correcto para login? (version/endpoint diferente?)
              echo  - TLS: si usas certificados reales, desactiva MI_TLS_INSEGURO=false y prueba con certificado correcto
              echo  - Revisa logs del Micro Integrator para trace del intent login
              echo -------------------------------------------------------

              exit /b 1
            )

            :parse_token
            REM Extraer AccessToken con PowerShell: escribir token ASCII a token.txt
            powershell -NoProfile -Command ^
              "try { $j = Get-Content 'login.json' -Raw | ConvertFrom-Json; if ($j -and $j.AccessToken) { $j.AccessToken | Out-File -Encoding ascii token.txt } else { exit 2 } } catch { exit 2 }"

            if %ERRORLEVEL% neq 0 (
              echo ERROR: No se pudo parsear login.json para extraer AccessToken.
              echo ==== login.json ====
              if exist login.json (
                type login.json
              ) else (
                echo login.json no creado
              )
              exit /b 1
            )

            set /p TOKEN=<token.txt
            if "%TOKEN%"=="" (
              echo ERROR: token vacío tras parseo.
              if exist login.json (
                type login.json
              )
              exit /b 1
            )

            echo Token obtenido (oculto).
            echo -------------------------------------------------------
            echo 2) Subir .car usando Bearer token
            echo APPS: %APPS%
            echo -------------------------------------------------------

            for %%F in ("%WORKSPACE%\\target\\*.car") do (
              echo Subiendo: %%~nxF

              if "%MI_TLS_INSEGURO%"=="true" (
                curl -k -f -sS -X POST "%APPS%" ^
                  -H "Authorization: Bearer %TOKEN%" ^
                  -H "Accept: application/json" ^
                  -F "file=@%%F" ^
                  --write-out "%%{http_code}" > upload_status.txt 2>upload_debug.txt
              ) else (
                curl -f -sS -X POST "%APPS%" ^
                  -H "Authorization: Bearer %TOKEN%" ^
                  -H "Accept: application/json" ^
                  -F "file=@%%F" ^
                  --write-out "%%{http_code}" > upload_status.txt 2>upload_debug.txt
              )

              if exist upload_status.txt (
                set /p UP_HTTP=<upload_status.txt
              ) else (
                set "UP_HTTP="
              )
              echo Upload HTTP status: %UP_HTTP%

              echo ==== upload_debug.txt ====
              if exist upload_debug.txt (
                type upload_debug.txt
              ) else (
                echo upload_debug.txt no creado
              )

              if not "%UP_HTTP%"=="200" if not "%UP_HTTP%"=="201" (
                echo ERROR: fallo subiendo %%~nxF (HTTP %UP_HTTP%)
                if exist upload_debug.txt type upload_debug.txt
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
              curl -k -s -o import_resp.json --write-out "%%{http_code}" -u "%APIM_USER%:%APIM_PASS%" -F "file=@%OAS_FILE%" "https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v1/apis/import-openapi" > import_status.txt
              if exist import_status.txt (
                set /p HTTP_CODE=<import_status.txt
              ) else (
                set "HTTP_CODE="
              )
              echo Import REST HTTP status: %HTTP_CODE%

              if not "%HTTP_CODE%"=="200" if not "%HTTP_CODE%"=="201" (
                echo ERROR: fallo importando via REST (HTTP %HTTP_CODE%)
                echo Contenido import_resp.json:
                if exist import_resp.json type import_resp.json
                exit /b 1
              )
              echo Import via REST completado.
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