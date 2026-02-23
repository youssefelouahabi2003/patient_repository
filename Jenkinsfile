pipeline {
  agent any

  parameters {
    string(name: 'MI_HOST', defaultValue: 'localhost', description: 'Host o IP donde está Micro Integrator', trim: true)
    string(name: 'MI_MGMT_PORT', defaultValue: '9164', description: 'Puerto Management API de MI', trim: true)
    booleanParam(name: 'MI_TLS_INSEGURO', defaultValue: true, description: 'Aceptar certificado TLS no confiable (dev)')
    booleanParam(name: 'COMPROBAR_HTTP', defaultValue: false, description: 'Comprobar endpoint runtime tras desplegar')
    string(name: 'MI_RUNTIME_PORT', defaultValue: '8290', description: 'Puerto runtime HTTP de MI', trim: true)
    string(name: 'HEALTH_PATH', defaultValue: '/patients/', description: 'Ruta a probar tras el despliegue', trim: true)

    // APIM: por defecto localhost
    string(name: 'APIM_HOST', defaultValue: 'localhost', description: 'Host API Manager', trim: true)
    string(name: 'APIM_PORT', defaultValue: '9443', description: 'Puerto API Manager', trim: true)

    // Opcionales para publicar automáticamente
    string(name: 'API_NAME', defaultValue: '', description: 'Nombre de la API (opcional para publish automático)', trim: true)
    string(name: 'API_VERSION', defaultValue: '', description: 'Versión de la API (opcional para publish automático)', trim: true)
  }

  environment {
    MI_HOST         = "${params.MI_HOST ?: 'localhost'}"
    MI_MGMT_PORT    = "${params.MI_MGMT_PORT ?: '9164'}"
    MI_TLS_INSEGURO = "${params.MI_TLS_INSEGURO}"
    MI_RUNTIME_PORT = "${params.MI_RUNTIME_PORT ?: '8290'}"
    HEALTH_PATH     = "${params.HEALTH_PATH ?: '/patients/'}"

    APIM_HOST       = "${params.APIM_HOST ?: 'localhost'}"
    APIM_PORT       = "${params.APIM_PORT ?: '9443'}"
    API_NAME        = "${params.API_NAME ?: ''}"
    API_VERSION     = "${params.API_VERSION ?: ''}"
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
        bat '''
          @echo off
          mvn -B -DskipTests clean package
        '''
      }
    }

    stage('Verificar .car') {
      steps {
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

    // Stage MI: sin tocar (tal como lo tenías)
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

    // Stage APIM: escribe y ejecuta apim_deploy.cmd (LOGIN + IMPORT con verbose y captura)
    stage('Publicar en API Manager (Windows)') {
      steps {
        script {
          def oasRelative = 'src\\main\\wso2mi\\resources\\api-definitions\\HealthcareAPI1.yaml'
          def oasPath = "${env.WORKSPACE}\\${oasRelative}"
          if (!fileExists(oasPath)) {
            echo "Aviso: no se encuentra ${oasPath}. Se omite publicación en APIM."
            return
          }

          // script .cmd corregido: añade login y usa --verbose en import
          def scriptContent = '''
@echo off
setlocal

set "APIM_HOST=%APIM_HOST%"
set "APIM_PORT=%APIM_PORT%"
set "APIM_USER=%APIM_USER%"
set "APIM_PASS=%APIM_PASS%"
set "API_NAME=%API_NAME%"
set "API_VERSION=%API_VERSION%"
set "OAS_FILE=%WORKSPACE%\\src\\main\\wso2mi\\resources\\api-definitions\\HealthcareAPI1.yaml"

echo ------------------------------------------
echo Publicación en API Manager - inicio
echo APIM host: %APIM_HOST%:%APIM_PORT%
echo OAS: %OAS_FILE%
echo ------------------------------------------

rem 1) comprobar si apictl está instalado
where apictl > NUL 2> NUL
if errorlevel 1 (
  echo apictl NO encontrado -> fallback REST import
  set "USE_APICTL=0"
) else (
  echo apictl encontrado
  set "USE_APICTL=1"
)

echo [DEBUG] USE_APICTL=%USE_APICTL%

rem 2) si apictl disponible -> comprobar/añadir env 'ci' y luego login/import
if "%USE_APICTL%"=="1" (
  echo [DEBUG] list envs
  apictl list env > apictl_envs.txt 2>apictl_envs_err.txt
  echo [DEBUG] contenido apictl_envs.txt:
  if exist apictl_envs.txt type apictl_envs.txt

  findstr /I /C:"ci" apictl_envs.txt >NUL
  if errorlevel 1 (
    echo Environment 'ci' no encontrado en apictl -> añadiendo...
    apictl add env ci --apim "https://%APIM_HOST%:%APIM_PORT%" --username "%APIM_USER%" --password "%APIM_PASS%" --insecure > apictl_add_env_out.txt 2> apictl_add_env_err.txt
    if errorlevel 1 (
      echo ERROR: fallo al añadir environment 'ci' en apictl.
      if exist apictl_add_env_err.txt type apictl_add_env_err.txt
      exit /b 1
    )
    echo Environment 'ci' añadido correctamente.
  ) else (
    echo Environment 'ci' ya existe en apictl.
  )

  rem 2b) login al env ci (necesario antes del import)
  echo [DEBUG] apictl login ci -u <user> -p <masked> -k
  apictl login ci -u "%APIM_USER%" -p "%APIM_PASS%" -k > apictl_login_out.txt 2> apictl_login_err.txt
  if errorlevel 1 (
    echo ERROR: apictl login falló.
    if exist apictl_login_err.txt type apictl_login_err.txt
    exit /b 1
  )
  echo apictl login OK.

  rem 3) Import usando apictl (con verbose)
  echo [DEBUG] apictl import api --verbose
  apictl import api -f "%OAS_FILE%" -e ci --update --verbose > apictl_import_out.txt 2> apictl_import_err.txt
  if errorlevel 1 (
    echo ERROR: apictl import api falló. Imprimiendo logs:
    if exist apictl_import_out.txt (echo ----- STDOUT ----- & type apictl_import_out.txt)
    if exist apictl_import_err.txt (echo ----- STDERR ----- & type apictl_import_err.txt)
    exit /b 1
  )
  echo apictl import api OK. Mostrando salida mínima:
  if exist apictl_import_out.txt type apictl_import_out.txt

  rem publicar si se indica (ifs anidados seguros)
  if not "%API_NAME%"=="" (
    if not "%API_VERSION%"=="" (
      echo Publicando API %API_NAME% %API_VERSION% con apictl...
      apictl change-status api -a Publish -n "%API_NAME%" -v "%API_VERSION%" -r "%APIM_USER%" -e ci > apictl_publish_out.txt 2>apictl_publish_err.txt
      if errorlevel 1 (
        echo ERROR: apictl change-status ha fallado.
        if exist apictl_publish_err.txt type apictl_publish_err.txt
        exit /b 1
      )
      echo API publicada correctamente con apictl.
    ) else (
      echo API_VERSION no proporcionada; salto publish.
    )
  ) else (
    echo API_NAME no proporcionada; salto publish.
  )

) else (
  rem fallback REST: import via publisher import-openapi
  echo apictl no disponible -> import via REST
  curl -k -s -o import_resp.json --write-out "%%{http_code}" -u "%APIM_USER%:%APIM_PASS%" -F "file=@%OAS_FILE%" "https://%APIM_HOST%:%APIM_PORT%/api/am/publisher/v1/apis/import-openapi" > import_status.txt 2> import_debug.txt

  if exist import_status.txt (
    set /p HTTP_CODE=<import_status.txt
  ) else (
    set "HTTP_CODE="
  )
  echo Import REST HTTP status: %HTTP_CODE%

  if not "%HTTP_CODE%"=="200" if not "%HTTP_CODE%"=="201" (
    echo ERROR: fallo importando via REST (HTTP %HTTP_CODE%)
    if exist import_resp.json type import_resp.json
    if exist import_debug.txt type import_debug.txt
    exit /b 1
  )
  echo Import via REST completado correctamente.
)

endlocal
'''
          writeFile file: "${env.WORKSPACE}\\apim_deploy.cmd", text: scriptContent
        }

        withCredentials([usernamePassword(credentialsId: 'APIM_ADMIN', usernameVariable: 'APIM_USER', passwordVariable: 'APIM_PASS')]) {
          // ejecuta el script que acabamos de crear
          bat "call \"%WORKSPACE%\\apim_deploy.cmd\""
        }
      }
    }

    stage('Comprobación HTTP (opcional)') {
      when { expression { return params.COMPROBAR_HTTP } }
      steps {
        bat '''
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
        '''
      }
    }
  } // stages

  post {
    success {
      archiveArtifacts artifacts: 'target/*.car', fingerprint: true
      echo 'Pipeline finalizado con éxito.'
    }
    failure {
      echo 'Pipeline fallido. Revisa la consola para detalles.'
    }
  }
}