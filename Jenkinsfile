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
    booleanParam(name: 'COMPROBAR_HTTP', defaultValue: true, description: 'Comprobar endpoint HTTP tras desplegar')
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
        // Cambia 'main' por 'master' si tu rama principal es master
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
        script {
          def listado = bat(returnStdout: true, script: 'if exist target\\*.car (dir /B target\\*.car) else (echo __NO_CAR__)').trim()
          if (listado == '__NO_CAR__') {
            error 'No se generó ningún .car en target\\'
          }
          echo "CAR(s) encontrados:\n${listado}"
        }
      }
    }

    stage('Desplegar en Micro Integrator por API (sin auth)') {
      steps {
        bat '''
          @echo off
          setlocal enabledelayedexpansion

          set "MI_URL=https://%MI_HOST%:%MI_MGMT_PORT%"
          set "ENDPOINT=%MI_URL%/management/applications"

          echo ------------------------------------------
          echo Subiendo .car a Micro Integrator por API (sin auth)
          echo Endpoint: %ENDPOINT%
          echo ------------------------------------------

          for %%F in ("%WORKSPACE%\\target\\*.car") do (
            echo Subiendo: %%~nxF

            if "%MI_TLS_INSEGURO%"=="true" (
              curl -k -s -X POST "%ENDPOINT%" -F "file=@%%F"
            ) else (
              curl -s -X POST "%ENDPOINT%" -F "file=@%%F"
            )

            if errorlevel 1 (
              echo ERROR: fallo subiendo %%~nxF
              exit /b 1
            )

            echo OK: %%~nxF subido
            echo.
          )

          echo Despliegue por API completado.
          exit /b 0
        '''
      }
    }
    stage('Comprobación HTTP (opcional)') {
      when { expression { return params.COMPROBAR_HTTP } }
      steps {
        echo 'Esperando 3s a que MI procese el .car...'
        sleep 3

        script {
          def intentos = 6
          def ok = false
          def url = "http://${params.MI_HOST}:${params.MI_RUNTIME_PORT}${params.HEALTH_PATH}"

          for (int i = 1; i <= intentos; i++) {
            try {
              bat(returnStdout: true, script: "powershell -NoProfile -Command \"(Invoke-WebRequest -UseBasicParsing -Uri '${url}' -TimeoutSec 5).StatusCode\"")
              ok = true
              break
            } catch (e) {
              echo "Intento ${i}/${intentos}: aún no responde ${url}. Esperamos 5s..."
              sleep 5
            }
          }

          if (!ok) {
            error "El endpoint no respondió en el tiempo esperado: ${url}"
          } else {
            echo "OK: el endpoint respondió: ${url}"
          }
        }
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