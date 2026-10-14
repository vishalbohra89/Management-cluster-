pipeline {
    agent {
        kubernetes {
            defaultContainer 'kubectl'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kubectl
    image: ghcr.io/cessda/jenkins-agent:1.0
    command:
    - cat
    tty: true
"""
        }
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        MANIFEST_DIR = 'cessda-mgmt'
        KUBECONFIG   = "${WORKSPACE}/.kube/config"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Configure kubeconfig') {
            steps {
                container('kubectl') {
                    withCredentials([string(credentialsId: 'cessda-mgmt-sa-token', variable: 'SA_TOKEN')]) {
                        sh '''
                            mkdir -p "$(dirname "$KUBECONFIG")"
                            kubectl config set-cluster cessda-mgmt \
                                --server="${CESSDA_MGMT_API_SERVER}" \
                                --insecure-skip-tls-verify=false \
                                --certificate-authority="${WORKSPACE}/ca.crt" 2>/dev/null || \
                            kubectl config set-cluster cessda-mgmt \
                                --server="${CESSDA_MGMT_API_SERVER}"
                            kubectl config set-credentials jenkins-deployer --token="${SA_TOKEN}"
                            kubectl config set-context cessda-mgmt \
                                --cluster=cessda-mgmt --user=jenkins-deployer --namespace=cessda-mgmt
                            kubectl config use-context cessda-mgmt
                        '''
                    }
                }
            }
        }

        stage('Validate (dry-run server)') {
            steps {
                container('kubectl') {
                    sh '''
                        set -e
                        echo "== RBAC =="
                        kubectl apply --dry-run=server -f "${MANIFEST_DIR}/rbac.yaml"
                        kubectl apply --dry-run=server -f "${MANIFEST_DIR}/rbac/"

                        echo "== LB IP pool =="
                        kubectl apply --dry-run=server -f "${MANIFEST_DIR}/pool.yaml"

                        echo "== ClusterIssuer =="
                        kubectl apply --dry-run=server -f "${MANIFEST_DIR}/clusterissuer-letsencrypt.yaml"

                        echo "== Certificate =="
                        kubectl apply --dry-run=server -f "${MANIFEST_DIR}/certificate.yaml"

                        echo "== Gateway =="
                        kubectl apply --dry-run=server -f "${MANIFEST_DIR}/gateway.yaml"

                        echo "== HTTPRoutes =="
                        kubectl apply --dry-run=server -f argoCD/httproute.yaml
                        kubectl apply --dry-run=server -f monitoring/httproute.yaml
                    '''
                }
            }
        }

        stage('Diff against live cluster') {
            steps {
                container('kubectl') {
                    sh '''
                        set +e
                        for f in "${MANIFEST_DIR}/rbac.yaml" "${MANIFEST_DIR}/pool.yaml" \
                                 "${MANIFEST_DIR}/clusterissuer-letsencrypt.yaml" \
                                 "${MANIFEST_DIR}/certificate.yaml" "${MANIFEST_DIR}/gateway.yaml" \
                                 argoCD/httproute.yaml monitoring/httproute.yaml; do
                            echo "---- diff: $f ----"
                            kubectl diff -f "$f" || true
                        done
                        for f in "${MANIFEST_DIR}/rbac/"*.yaml; do
                            echo "---- diff: $f ----"
                            kubectl diff -f "$f" || true
                        done
                        exit 0
                    '''
                }
            }
        }

        stage('Approval') {
            steps {
                script {
                    input message: "Apply the above changes to cessda-mgmt?", ok: "Apply"
                }
            }
        }

        stage('Apply RBAC') {
            steps {
                container('kubectl') {
                    sh '''
                        kubectl apply -f "${MANIFEST_DIR}/rbac.yaml"
                        kubectl apply -f "${MANIFEST_DIR}/rbac/"
                    '''
                }
            }
        }

        stage('Apply LB IP pool') {
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f "${MANIFEST_DIR}/pool.yaml"'
                }
            }
        }

        stage('Apply ClusterIssuer') {
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f "${MANIFEST_DIR}/clusterissuer-letsencrypt.yaml"'
                }
            }
        }

        stage('Apply Certificate') {
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f "${MANIFEST_DIR}/certificate.yaml"'
                }
            }
        }

        stage('Apply Gateway') {
            steps {
                container('kubectl') {
                    sh '''
                        kubectl apply -f "${MANIFEST_DIR}/gateway.yaml"
                        kubectl wait --for=condition=Accepted gateway/cessda-gateway \
                            -n cessda-mgmt --timeout=60s
                    '''
                }
            }
        }

        stage('Apply HTTPRoutes') {
            steps {
                container('kubectl') {
                    sh '''
                        kubectl apply -f argoCD/httproute.yaml
                        kubectl apply -f monitoring/httproute.yaml
                    '''
                }
            }
        }

        stage('Post-deploy verification') {
            steps {
                container('kubectl') {
                    sh '''
                        echo "== Gateway status =="
                        kubectl get gateway cessda-gateway -n cessda-mgmt -o wide

                        echo "== Certificate status =="
                        kubectl get certificate cessda-tls -n cessda-mgmt -o wide

                        echo "== HTTPRoutes =="
                        kubectl get httproute -A

                        echo "== RBAC in cilium-secrets =="
                        kubectl get role,rolebinding -n cilium-secrets
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "cessda-mgmt manifests applied successfully."
        }
        failure {
            echo "Deployment failed. Check the stage logs above. No automatic rollback is performed — review 'kubectl diff' output before re-running."
        }
        always {
            container('kubectl') {
                sh 'rm -f "$KUBECONFIG" || true'
            }
        }
    }
}
