pipeline {
  agent any

  tools {
    jdk   'JDK17'
    maven 'Maven3'
  }

  parameters {
    string(name: 'MI_HOST', defaultValue: 'localhost', description: 'Host o IP donde está Micro Integrator')
    string(name: 'MI_MGMT_PORT', defaultValue: '9164', description: 'Puerto de Management API de MI')
    booleanParam(name: 'MI_TLS_INSEGURO', defaultValue: true, description: 'Aceptar certificado TLS no confiable (dev)')
    booleanParam(name: 'COMPROBAR_HTTP', defaultValue: false, description: 'Comprobar endpoint HTTP tras desplegar')
    string(name: 'MI_RUNTIME_PORT', defaultValue: '8290', description: 'Puerto runtime HTTP de MI')
    string(name: 'HEALTH_PATH', defaultValue: '/patients/', description: 'Ruta a probar tras el despliegue')
  }

  options {
    timestamps()
    timeout(time: 45, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/youssefelouahabi2003/patient_repository.git'
      }
    }

    stage('Build (Maven)') {
      steps {
        script {
          if (isUnix()) {
            sh 'mvn -B -DskipTests clean package'
          } else {
            bat 'mvn -B -DskipTests clean package'
          }
        }
      }
    }

    stage('Verificar .car') {
      steps {
        script {
          if (isUnix()) {
            def listado = sh(returnStdout: true, script: 'if ls target/*.car 1> /dev/null 2>&1; then ls target/*.car; else echo __NO_CAR__; fi').trim()
            if (listado == '__NO_CAR__') error('No se generó ningún .car en target/')
            echo "CAR(s) encontrados:\n${listado}"
          } else {
            def listado = bat(returnStdout: true, script: 'if exist target\\*.car (dir /B target\\*.car) else (echo __NO_CAR__)').trim()
            if (listado == '__NO_CAR__') error('No se generó ningún .car en target\\')
            echo "CAR(s) encontrados:\n${listado}"
          }
        }
      }
    }

    stage('Desplegar en Micro Integrator por API') {
      steps {
        // Usa credenciales de Jenkins en lugar de hardcodear
        withCredentials([usernamePassword(credentialsId: 'MI_ADMIN', usernameVariable: 'MI_USER', passwordVariable: 'MI_PASS')]) {
          script {
            def baseUrl = "https://${params.MI_HOST}:${params.MI_MGMT_PORT}/management"
            def loginUrl = "${baseUrl}/login"
            def appsUrl = "${baseUrl}/applications"
            def curl_k = params.MI_TLS_INSEGURO ? "-k" : ""

            if (isUnix()) {
              sh """
                set -euo pipefail
                echo "Comprobando curl..."
                if ! command -v curl >/dev/null 2>&1; then echo "ERROR: curl no encontrado"; exit 1; fi

                echo "1) LOGIN ${loginUrl}"
                HTTP_LOGIN=\$(curl ${curl_k} -sS -w "%{http_code}" -u "${env.MI_USER}:${env.MI_PASS}" "${loginUrl}" -o login.json)
                echo "login HTTP status: \$HTTP_LOGIN"
                if [ "\$HTTP_LOGIN" -ne 200 ] && [ "\$HTTP_LOGIN" -ne 201 ]; then
                  echo "Respuesta login:"
                  cat login.json || true
                  exit 1
                fi

                ACCESS_TOKEN=\$(python3 -c "import sys, json; print(json.load(open('login.json'))['AccessToken'])" 2>/dev/null || python -c "import sys,json; print(json.load(open('login.json'))['AccessToken'])")
                if [ -z "\$ACCESS_TOKEN" ]; then echo "No se obtuvo AccessToken"; cat login.json; exit 1; fi
                echo "Token obtenido."

                for f in target/*.car; do
                  echo "Subiendo \$f..."
                  HTTP_UPLOAD=\$(curl ${curl_k} -sS -o /dev/null -w "%{http_code}" -X POST "${appsUrl}" -H "Authorization: Bearer \$ACCESS_TOKEN" -H "Accept: application/json" -F "file=@\$f")
                  echo "Upload status: \$HTTP_UPLOAD"
                  if [ "\$HTTP_UPLOAD" -lt 200 ] || [ "\$HTTP_UPLOAD" -ge 300 ]; then
                    echo "Fallo al subir \$f (HTTP \$HTTP_UPLOAD)"; exit 1
                  fi
                done

                echo "Despliegue por API completado (Unix)."
              """
            } else {
              // Windows (bat). No usar delayed expansion antes de asignar MI_PASS.
              bat """
                @echo off
                set "MI_HOST=${params.MI_HOST}"
                set "MI_MGMT_PORT=${params.MI_MGMT_PORT}"
                set "MI_USER=%MI_USER%"
                set "MI_PASS=%MI_PASS%"

                set "BASE=https://%MI_HOST%:%MI_MGMT_PORT%/management"
                set "LOGIN=%BASE%/login"
                set "APPS=%BASE%/applications"

                REM comprueba curl
                where curl >nul 2>nul || (echo ERROR: curl no encontrado & exit /b 1)

                echo 1) LOGIN %LOGIN%
                if "${params.MI_TLS_INSEGURO}"=="true" (
                  curl -k -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json -w "HTTP_STATUS=%{http_code}"
                ) else (
                  curl -sS -u "%MI_USER%:%MI_PASS%" "%LOGIN%" -o login.json -w "HTTP_STATUS=%{http_code}"
                )
                type login.json

                for /f "usebackq delims=" %%T in (`powershell -NoProfile -Command "(Get-Content login.json -Raw | ConvertFrom-Json).AccessToken"`) do set "TOKEN=%%T"

                if "%TOKEN%"=="" (
                  echo ERROR: No se pudo obtener token.
                  exit /b 1
                )

                echo Token obtenido.

                for %%F in ("%WORKSPACE%\\target\\*.car") do (
                  echo Subiendo: %%~nxF
                  if "${params.MI_TLS_INSEGURO}"=="true" (
                    curl -k -sS -X POST "%APPS%" -H "Authorization: Bearer %TOKEN%" -H "Accept: application/json" -F "file=@%%F" -w "HTTP_STATUS=%{http_code}"
                  ) else (
                    curl -sS -X POST "%APPS%" -H "Authorization: Bearer %TOKEN%" -H "Accept: application/json" -F "file=@%%F" -w "HTTP_STATUS=%{http_code}"
                  )
                  if errorlevel 1 (
                    echo ERROR subiendo %%~nxF
                    exit /b 1
                  )
                )

                echo Despliegue por API completado (Windows).
              """
            } // isUnix else end
          } // script end
        } // withCredentials end
      } // steps end
    } // stage end

    stage('Comprobación HTTP (opcional)') {
      when { expression { return params.COMPROBAR_HTTP } }
      steps {
        script {
          def url = "http://${params.MI_HOST}:${params.MI_RUNTIME_PORT}${params.HEALTH_PATH}"
          def intentos = 12
          def ok = false

          for (int i=1; i<=intentos; i++) {
            try {
              if (isUnix()) {
                sh(returnStdout: true, script: "curl -sS -o /dev/null -w '%{http_code}' -m 8 ${url}")
              } else {
                bat(returnStdout: true, script: "powershell -NoProfile -Command \"(Invoke-WebRequest -UseBasicParsing -Uri '${url}' -TimeoutSec 8).StatusCode\"")
              }
              ok = true
              break
            } catch (e) {
              echo "Intento ${i}/${intentos}: aún no responde ${url}. Esperamos 5s..."
              sleep 5
            }
          }

          if (!ok) error "El endpoint no respondió: ${url}"
          echo "OK: respondió ${url}"
        }
      }
    }
  } // stages end

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