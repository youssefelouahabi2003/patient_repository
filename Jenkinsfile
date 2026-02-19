pipeline {
  agent any   // o agent { label 'windows' } si tu agente es Windows con etiqueta

  tools {
    jdk   'JDK17'    // Deben existir en "Administrar Jenkins -> Configuración global de herramientas"
    maven 'Maven3'
  }

  environment {
    MI_DEST = 'C:\\opt\\micro-integrator\\wso2mi-4.3.0.21\\wso2mi-4.3.0\\repository\\deployment\\server\\carbonapps'
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

    stage('Copiar a Micro Integrator') {
      steps {
        bat '''
          @echo off
          if not exist "%MI_DEST%" mkdir "%MI_DEST%"
          rem Copiamos los .car y controlamos el exit code de robocopy
          robocopy "%WORKSPACE%\\target" "%MI_DEST%" *.car /NFL /NDL /NJH /NJS /COPY:DAT
          set RC=%ERRORLEVEL%
          echo robocopy rc=%RC%
          if %RC% GEQ 8 (
            echo ERROR: robocopy devolvió %RC%
            exit /b 1
          )
          echo CAR copiado a: %MI_DEST%
        '''
      }
    }

    // (Opcional) comprobación rápida del endpoint
    stage('Comprobación HTTP (opcional)') {
      when { expression { return true } }  // pon false si no quieres comprobar
      steps {
        echo 'Esperando 3s a que MI procese el .car...'
        sleep 3
        script {
          def intentos = 6
          def ok = false
          for (int i=1; i<=intentos; i++) {
            try {
              bat(returnStdout: true, script: 'powershell -NoProfile -Command "(Invoke-WebRequest -UseBasicParsing -Uri \\"http://localhost:8290/patients/\\" -TimeoutSec 5).StatusCode"')
              ok = true; break
            } catch (e) {
              echo "Intento ${i}/${intentos}: aún no responde. Esperamos 5s..."
              sleep 5
            }
          }
          if (!ok) error 'El endpoint no respondió en el tiempo esperado.'
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
