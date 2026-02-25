pipeline {
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

  environment {
    MI_HOST = "${params.MI_HOST}"
    MI_MGMT_PORT = "${params.MI_MGMT_PORT}"
    MI_RUNTIME_PORT = "${params.MI_RUNTIME_PORT}"
    HEALTH_PATH = "${params.HEALTH_PATH}"
    MI_TLS_INSEGURO = "${params.MI_TLS_INSEGURO}"

    APIM_HOST = "localhost"
    APIM_PORT = "9443"
    API_NAME = "pepeprueba"
    API_VERSION = "1.0.0"
    API_CONTEXT = "/pepedoctor"
    BACKEND_URL = "http://localhost:8290"
    REPO_OAS = "${WORKSPACE}/src/main/wso2mi/resources/api-definitions/HealthcareAPI1.yaml"
    CAR_EXTRACT = "${WORKSPACE}/car_extract"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: 'main']], userRemoteConfigs: [[url: 'https://github.com/youssefelouahabi2003/patient_repository.git']]])
      }
    }

    stage('Build (Maven)') {
      steps {
        sh 'mvn -B -DskipTests clean package'
      }
    }

    stage('Verificar .car') {
      steps {
        sh '''
          set -e
          echo "Buscando .car en target/ ..."
          shopt -s nullglob
          cars=(target/*.car)
          if [ ${#cars[@]} -gt 0 ]; then
            echo "CARs encontrados:"
            for c in "${cars[@]}"; do echo "$c"; done
          else
            echo "ERROR: No se generó ningún .car en target/"
            exit 1
          fi
        '''
      }
    }

    stage('Desplegar en Micro Integrator (Linux)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'MI_ADMIN', usernameVariable: 'MI_USER', passwordVariable: 'MI_PASS')]) {
          sh '''
            set -euo pipefail

            BASE="https://${MI_HOST}:${MI_MGMT_PORT}/management"
            LOGIN="${BASE}/login"
            APPS="${BASE}/applications"

            if [ "${MI_TLS_INSEGURO}" = "true" ]; then CURL_TLS_OPT="-k"; else CURL_TLS_OPT=""; fi

            echo "1) Login MI -> ${LOGIN}"
            curl ${CURL_TLS_OPT} -sS -u "${MI_USER}:${MI_PASS}" "${LOGIN}" -o login.json || (cat login.json || true; echo "ERROR: fallo login MI"; exit 1)

            # Extraer AccessToken con awk (no backslashes problemáticos)
            MI_TOKEN=$(awk 'BEGIN{token="";} { if(match($0,/"AccessToken"[[:space:]]*:[[:space:]]*"[^\"]*"/)) { if(match($0,/"AccessToken"[[:space:]]*:[[:space:]]*"([^"]*)"/,a)) { print a[1]; exit } } }' login.json || true)
            if [ -z "${MI_TOKEN}" ]; then
              echo "ERROR: no se obtuvo AccessToken de MI. login.json:"
              sed -n '1,200p' login.json || true
              exit 1
            fi
            echo "Token MI (preview): ${MI_TOKEN:0:40}..."

            # Subir cada .car
            for car in target/*.car; do
              echo "Subiendo: ${car}"
              curl ${CURL_TLS_OPT} -f -sS -X POST "${APPS}" \
                -H "Authorization: Bearer ${MI_TOKEN}" \
                -H "Accept: application/json" \
                -F "file=@${car}" -o mi_deploy_resp.json || ( echo "ERROR: fallo subiendo ${car}"; sed -n '1,200p' mi_deploy_resp.json || true; exit 1 )
              echo "Respuesta MI upload (preview):"
              sed -n '1,200p' mi_deploy_resp.json || true
            done

            echo "Despliegue MI completado."
          '''
        }
      }
    }

    stage('Publicar API desde Swagger (.car or repo)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'APIM_ADMIN', usernameVariable: 'APIM_USER', passwordVariable: 'APIM_PASS')]) {
          sh '''
            set -euo pipefail
            if [ "${MI_TLS_INSEGURO}" = "true" ]; then CURL_TLS_OPT="-k"; else CURL_TLS_OPT=""; fi

            # seleccionar swagger (.car preferido)
            rm -rf "${CAR_EXTRACT}" || true
            car_file=$(ls target/*.car 2>/dev/null | head -n1 || true)
            FOUND_SWAGGER=""
            if [ -n "${car_file}" ]; then
              echo ".car encontrado: ${car_file}"
              mkdir -p "${CAR_EXTRACT}"
              unzip -q -o "${car_file}" -d "${CAR_EXTRACT}"
              FOUND_SWAGGER=$(find "${CAR_EXTRACT}" -type f \\( -iname "*.yaml" -o -iname "*.yml" -o -iname "*.json" \\) | head -n1 || true)
            fi

            if [ -z "${FOUND_SWAGGER}" ]; then
              if [ -f "${REPO_OAS}" ]; then
                FOUND_SWAGGER="${REPO_OAS}"
                echo "Usando swagger del repo: ${FOUND_SWAGGER}"
              else
                echo "ERROR: No se encontró swagger en .car ni en repo (${REPO_OAS})"
                exit 1
              fi
            else
              echo "Swagger desde .car: ${FOUND_SWAGGER}"
            fi

            # 1) DCR
            DCR_URL="https://${APIM_HOST}:${APIM_PORT}/client-registration/v0.17/register"
            echo "DCR_URL: ${DCR_URL}"
            # crear payload minimal sin BOM
            printf '%s' '{"callbackUrl":"http://localhost","clientName":"jenkins_publisher_api","tokenScope":"Production","owner":"'${APIM_USER}'","grantType":"password refresh_token","saasApp":true}' > dcr_payload.json

            curl ${CURL_TLS_OPT} -sS -u "${APIM_USER}:${APIM_PASS}" -H "Content-Type: application/json" --data-binary @dcr_payload.json "${DCR_URL}" -o dcr.json || ( echo "ERROR: DCR fallo"; sed -n '1,200p' dcr.json || true; exit 1 )

            CLIENT_ID=$(awk 'match($0,/"clientId"[[:space:]]*:[[:space:]]*"([^"]*)"/,a){print a[1]; exit}' dcr.json || true)
            CLIENT_SECRET=$(awk 'match($0,/"clientSecret"[[:space:]]*:[[:space:]]*"([^"]*)"/,a){print a[1]; exit}' dcr.json || true)

            if [ -z "${CLIENT_ID}" ] || [ -z "${CLIENT_SECRET}" ]; then
              echo "ERROR: DCR no devolvió clientId/clientSecret:"
              sed -n '1,200p' dcr.json || true
              exit 1
            fi
            echo "DCR OK (clientId preview): ${CLIENT_ID:0:8}..."

            # 2) Token OAuth2 (password grant)
            TOKEN_URL="https://${APIM_HOST}:${APIM_PORT}/oauth2/token"
            echo "TOKEN_URL: ${TOKEN_URL}"
            curl ${CURL_TLS_OPT} -sS -u "${CLIENT_ID}:${CLIENT_SECRET}" -H "Content-Type: application/x-www-form-urlencoded" \
              --data-urlencode "grant_type=password" \
              --data-urlencode "username=${APIM_USER}" \
              --data-urlencode "password=${APIM_PASS}" \
              --data-urlencode "scope=apim:api_view apim:api_create apim:api_manage" \
              "${TOKEN_URL}" -o apim_token.json || ( echo "ERROR: token fallo"; sed -n '1,200p' apim_token.json || true; exit 1 )

            APIM_TOKEN=$(awk 'match($0,/"access_token"[[:space:]]*:[[:space:]]*"([^"]*)"/,a){print a[1]; exit}' apim_token.json || true)
            if [ -z "${APIM_TOKEN}" ]; then
              echo "ERROR: access_token vacio"
              sed -n '1,200p' apim_token.json || true
              exit 1
            fi
            echo "Token APIM obtenido (preview): ${APIM_TOKEN:0:40}..."

            # 3) Import OpenAPI (crear API)
            PUB_BASE="https://${APIM_HOST}:${APIM_PORT}/api/am/publisher/v4"
            IMPORT_URL="${PUB_BASE}/apis/import-openapi"

            printf '%s' '{"name":"'"${API_NAME}"'","version":"'"${API_VERSION}"'","context":"'"${API_CONTEXT}"'"}' > additional.json
            echo "--- additional.json (debug) ---"
            sed -n '1,200p' additional.json || true
            echo

            echo "POST import-openapi -> ${IMPORT_URL}"
            curl ${CURL_TLS_OPT} -sS -X POST "${IMPORT_URL}" -H "Authorization: Bearer ${APIM_TOKEN}" -H "Accept: application/json" \
              -F "file=@${FOUND_SWAGGER}" -F "additionalProperties=@additional.json" -o created_api.json || true

            echo "--- created_api.json (debug) ---"
            sed -n '1,200p' created_api.json || true
            echo

            API_ID=$(awk 'match($0,/"id"[[:space:]]*:[[:space:]]*"([^"]*)"/,a){print a[1]; exit}' created_api.json || true)
            if [ -z "${API_ID}" ]; then
              echo "ERROR: import-openapi no devolvió id; ver created_api.json"
              exit 1
            fi
            echo "API creada. ID=${API_ID}"

            # 4) Update endpointConfig (PUT /apis/{apiId})
            printf '%s' '{"endpointConfig":{"endpoint_type":"http","production_endpoints":{"url":"'"${BACKEND_URL}"'"},"sandbox_endpoints":{"url":"'"${BACKEND_URL}"'"}}}' > update_body.json

            echo "PUT update endpoint -> ${PUB_BASE}/apis/${API_ID}"
            curl ${CURL_TLS_OPT} -sS -X PUT "${PUB_BASE}/apis/${API_ID}" -H "Authorization: Bearer ${APIM_TOKEN}" -H "Content-Type: application/json" -d @update_body.json -o update_api_result.json || ( echo "ERROR: fallo update endpoint"; sed -n '1,200p' update_api_result.json || true; exit 1 )

            echo "Endpoint actualizado OK."
          '''
        }
      }
    }

    stage('Comprobación HTTP (opcional)') {
      when { expression { return params.COMPROBAR_HTTP } }
      steps {
        sh '''
          set -euo pipefail
          echo "Esperando 10s a que MI procese el .car..."
          sleep 10
          URL="http://${MI_HOST}:${MI_RUNTIME_PORT}${HEALTH_PATH}"
          ATTEMPTS=12
          I=1
          while [ $I -le $ATTEMPTS ]; do
            if [ "${MI_TLS_INSEGURO}" = "true" ]; then
              out=$(curl -sS -o /dev/null -w "%{http_code}" -m 8 -k "$URL" 2>/dev/null || echo "000")
            else
              out=$(curl -sS -o /dev/null -w "%{http_code}" -m 8 "$URL" 2>/dev/null || echo "000")
            fi
            if [ "$out" != "000" ]; then
              echo "OK: respondió $URL -> $out"
              exit 0
            fi
            echo "Intento $I/$ATTEMPTS: aún no responde $URL"
            I=$((I+1))
            sleep 5
          done
          echo "El endpoint no respondió: $URL"
          exit 1
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
      echo 'Pipeline fallido. Revisa la consola.'
    }
  }
}