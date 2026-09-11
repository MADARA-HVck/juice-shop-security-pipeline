pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                echo '=== Code source récupéré depuis GitHub ==='

                sh '''
                    echo "Workspace : $PWD"
                    echo "Commit    : $(git rev-parse --short HEAD)"
                    echo "Branche   : $(git rev-parse --abbrev-ref HEAD)"
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
                    curl -fsS -I http://127.0.0.1:3000 > /tmp/juice-shop-http.txt
                    cat /tmp/juice-shop-http.txt
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

const vulnerabilities = data.metadata?.vulnerabilities || {};

console.log(`Low      : ${vulnerabilities.low ?? 0}`);
console.log(`Moderate : ${vulnerabilities.moderate ?? 0}`);
console.log(`High     : ${vulnerabilities.high ?? 0}`);
console.log(`Critical : ${vulnerabilities.critical ?? 0}`);
console.log(`Total    : ${vulnerabilities.total ?? 0}`);
NODE
                '''
            }
        }

        stage('Additional Security Check - DAST') {
            steps {
                echo '=== Analyse DAST avec OWASP ZAP ==='

                sh '''
                    mkdir -p reports
                    chmod 777 reports

                    echo "Lancement de ZAP Baseline Scan..."

                    docker run --rm \
                        --network host \
                        -v "$PWD/reports:/zap/wrk/:rw" \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://127.0.0.1:3000 \
                        -J dast.json \
                        -r dast.html \
                        || true

                    echo
                    echo "=== Vérification des rapports ZAP ==="

                    test -f reports/dast.json
                    test -f reports/dast.html

                    ls -lh reports/
                '''
            }
        }

        stage('Report Generation') {
            steps {
                echo '=== Génération du résumé de sécurité ==='

                sh '''
                    {
                        echo "=========================================="
                        echo "   JUICE SHOP - SECURITY PIPELINE"
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
                            echo "Rapport JSON ZAP : présent"
                        else
                            echo "Rapport JSON ZAP : absent"
                        fi

                        if [ -f reports/dast.html ]; then
                            echo "Rapport HTML ZAP : présent"
                        else
                            echo "Rapport HTML ZAP : absent"
                        fi

                        echo
                        echo "=========================================="
                    } > reports/security-summary.txt

                    cat reports/security-summary.txt
                '''
            }
        }

        stage('Notification') {
            steps {
                echo '=== Notification ==='
                echo "Le pipeline #${BUILD_NUMBER} est terminé."
                echo "Les rapports sont disponibles dans les artefacts Jenkins."
            }
        }
    }

    post {

        always {
            echo '=== Archivage des rapports ==='

            archiveArtifacts artifacts: 'reports/**',
                             allowEmptyArchive: false,
                             fingerprint: true
        }

        success {
            echo '=== Pipeline terminé avec succès ==='
        }

        failure {
            echo '=== Pipeline terminé avec échec ==='
        }
    }
}
