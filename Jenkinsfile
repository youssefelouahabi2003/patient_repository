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

    stage('Desplegar en Micro Integrator por API (Basic Auth)') {
  steps {
    withCredentials([usernamePassword(credentialsId: 'MI_ADMIN', usernameVariable: 'MI_USER', passwordVariable: 'MI_PASS')]) {
      bat """
        @echo off
        setlocal enabledelayedexpansion

        set "MI_HOST=${params.MI_HOST}"
        set "MI_MGMT_PORT=${params.MI_MGMT_PORT}"

        if "%MI_HOST%"=="" (
          echo ERROR: MI_HOST vacio
          exit /b 1
        )
        if "%MI_MGMT_PORT%"=="" (
          echo ERROR: MI_MGMT_PORT vacio
          exit /b 1
        )

        set "ENDPOINT=https://%MI_HOST%:%MI_MGMT_PORT%/management/applications"

        echo ------------------------------------------
        echo Subiendo .car a Micro Integrator por API (Basic Auth)
        echo Endpoint: %ENDPOINT%
        echo ------------------------------------------

        for %%F in ("%WORKSPACE%\\target\\*.car") do (
          echo Subiendo: %%~nxF

          if "${params.MI_TLS_INSEGURO}"=="true" (
            curl -k -f -sS -X POST "%ENDPOINT%" ^
              -u "%MI_USER%:%MI_PASS%" ^
              -H "Accept: application/json" ^
              -F "file=@%%F" ^
              -w "\\nHTTP_STATUS=%%{http_code}\\n"
          ) else (
            curl -f -sS -X POST "%ENDPOINT%" ^
              -u "%MI_USER%:%MI_PASS%" ^
              -H "Accept: application/json" ^
              -F "file=@%%F" ^
              -w "\\nHTTP_STATUS=%%{http_code}\\n"
          )

          if errorlevel 1 (
            echo ERROR: curl fallo subiendo %%~nxF
            exit /b 1
          )

          echo OK: %%~nxF subido
          echo.
        )

        echo Despliegue por API completado.
        exit /b 0
      """
    }
  }
}

    stage('Comprobación HTTP (opcional)') {
      when { expression { return params.COMPROBAR_HTTP } }
      steps {
        echo 'Esperando 3s a que MI procese el .car...'
        sleep 3

        script {
          def url = "http://${params.MI_HOST}:${params.MI_RUNTIME_PORT}${params.HEALTH_PATH}"
          def intentos = 6
          def ok = false

          for (int i=1; i<=intentos; i++) {
            try {
              bat(returnStdout: true, script: "powershell -NoProfile -Command \"(Invoke-WebRequest -UseBasicParsing -Uri '${url}' -TimeoutSec 5).StatusCode\"")
              ok = true
              break
            } catch (e) {
              echo "Intento ${i}/${intentos}: aún no responde ${url}. Esperamos 5s..."
              sleep 5
            }
          }

          if (!ok) error "El endpoint no respondió en el tiempo esperado: ${url}"
          echo "OK: el endpoint respondió: ${url}"
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