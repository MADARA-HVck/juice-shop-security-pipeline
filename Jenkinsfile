pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timeout(time: 25, unit: 'MINUTES')
    }

    stages {

        stage('Checkout') {
            steps {
                echo '=== Code source récupéré depuis GitHub ==='

                sh '''
                    echo "Workspace : $PWD"
                    echo "Commit    : $(git rev-parse --short HEAD)"
                    echo "Remote    : $(git remote get-url origin)"
                '''
            }
        }

        stage('Build / Preparation') {
            steps {
                echo '=== Vérification de l environnement ==='

                sh '''
                    echo "--- Node.js ---"
                    node --version

                    echo "--- npm ---"
                    npm --version

                    echo "--- Git ---"
                    git --version

                    echo "--- Docker ---"
                    docker --version

                    echo "--- Juice Shop ---"
                    curl -fsS -I --connect-timeout 10 http://127.0.0.1:3000
                '''
            }
        }

        stage('Security Analysis - SCA') {
            steps {
                echo '=== Analyse SCA avec npm audit ==='

                sh '''
                    mkdir -p reports

                    echo "Lancement de npm audit..."
                    npm audit --json > reports/sca.json || true

                    echo
                    echo "=== Résumé SCA ==="

                    node <<'NODE'
const fs = require('fs');

const data = JSON.parse(
    fs.readFileSync('reports/sca.json', 'utf8')
);

const v = data.metadata?.vulnerabilities || {};

console.log(`Low      : ${v.low ?? 0}`);
console.log(`Moderate : ${v.moderate ?? 0}`);
console.log(`High     : ${v.high ?? 0}`);
console.log(`Critical : ${v.critical ?? 0}`);
console.log(`Total    : ${v.total ?? 0}`);
NODE
                '''
            }
        }

        stage('Additional Security Check - DAST') {
            options {
                timeout(time: 15, unit: 'MINUTES')
            }

            steps {
                echo '=== Analyse DAST avec OWASP ZAP ==='

                sh '''
                    set +e

                    mkdir -p reports

                    ZAP_CONTAINER="zap-jenkins-${BUILD_NUMBER}"

                    cleanup() {
                        echo "Nettoyage du conteneur ZAP..."
                        docker rm -f "$ZAP_CONTAINER" >/dev/null 2>&1 || true
                    }

                    trap cleanup EXIT INT TERM

                    echo "Lancement de ZAP Baseline Scan..."
                    echo "Conteneur : $ZAP_CONTAINER"
                    echo "Cible     : http://127.0.0.1:3000"

                    timeout --foreground 12m \
                    docker run \
                        --name "$ZAP_CONTAINER" \
                        --network host \
                        -v "$PWD/reports:/zap/wrk/:rw" \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://127.0.0.1:3000 \
                        -J dast.json \
                        -r dast.html

                    ZAP_RC=$?

                    echo
                    echo "Code retour ZAP : $ZAP_RC"

                    if [ "$ZAP_RC" -eq 124 ]; then
                        echo "ERREUR : ZAP a dépassé la limite de 12 minutes."
                        exit 1
                    fi

                    echo "Le scan ZAP est terminé."

                    echo
                    echo "=== Vérification des rapports ZAP ==="

                    if [ ! -f reports/dast.json ]; then
                        echo "ERREUR : dast.json absent."
                        exit 1
                    fi

                    if [ ! -f reports/dast.html ]; then
                        echo "ERREUR : dast.html absent."
                        exit 1
                    fi

                    ls -lh reports/dast.json reports/dast.html

                    exit 0
                '''
            }
        }

        stage('Report Generation') {
            steps {
                echo '=== Génération du résumé de sécurité ==='

                sh '''
                    {
                        echo "=========================================="
                        echo "       JUICE SHOP SECURITY PIPELINE"
                        echo "=========================================="
                        echo
                        echo "Build Jenkins : #${BUILD_NUMBER}"
                        echo "Commit        : $(git rev-parse --short HEAD)"
                        echo "Date          : $(date)"
                        echo

                        echo "------------- SCA / npm audit ------------"

                        node <<'NODE'
const fs = require('fs');

const data = JSON.parse(
    fs.readFileSync('reports/sca.json', 'utf8')
);

const v = data.metadata?.vulnerabilities || {};

console.log(`Low      : ${v.low ?? 0}`);
console.log(`Moderate : ${v.moderate ?? 0}`);
console.log(`High     : ${v.high ?? 0}`);
console.log(`Critical : ${v.critical ?? 0}`);
console.log(`Total    : ${v.total ?? 0}`);
NODE

                        echo
                        echo "------------- DAST / OWASP ZAP -----------"

                        if [ -f reports/dast.json ]; then
                            echo "Rapport JSON ZAP  : OK"
                        else
                            echo "Rapport JSON ZAP  : ABSENT"
                        fi

                        if [ -f reports/dast.html ]; then
                            echo "Rapport HTML ZAP  : OK"
                        else
                            echo "Rapport HTML ZAP  : ABSENT"
                        fi

                        echo
                        echo "------------- Fichiers générés -----------"

                        ls -lh reports/

                        echo
                        echo "=========================================="
                    } > reports/security-summary.txt

                    cat reports/security-summary.txt
                '''
            }
        }
    }

    post {

        always {
            echo '=== Archivage des rapports ==='

            archiveArtifacts(
                artifacts: 'reports/**',
                allowEmptyArchive: true,
                fingerprint: true
            )

            echo '=== Envoi de la notification e-mail ==='

            mail(
                to: 'blackpower44444@gmail.com',
                subject: "Jenkins - Juice Shop Security Pipeline - Build #${BUILD_NUMBER} - ${currentBuild.currentResult}",
                body: """Bonjour,

Le pipeline de sécurité Juice Shop vient de se terminer.

Build : #${BUILD_NUMBER}
Statut : ${currentBuild.currentResult}
Commit : ${env.GIT_COMMIT ?: 'N/A'}

Rapports générés :
- SCA : npm audit
- DAST : OWASP ZAP
- security-summary.txt

Les rapports sont disponibles dans Jenkins.

URL du build :
${env.BUILD_URL}

Cordialement,
Jenkins
"""
            )
        }

        success {
            echo '=== Pipeline terminé avec succès ==='
        }

        failure {
            echo '=== Pipeline terminé avec échec ==='
        }

        aborted {
            echo '=== Pipeline interrompu ==='
        }
    }
}
