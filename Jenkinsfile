// ============================================================
//  Jenkinsfile — Terraform Pipeline with Environment Choice
//  Select ENV → Init → Plan → Approve → Apply
// ============================================================

pipeline {

    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'uat', 'prod'],
            description: 'Select the environment to run Terraform against'
        )
    }

    environment {
   //     AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
   //     AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'us-east-1'
        TF_VAR_FILE           = "envs/${params.ENVIRONMENT}.tfvars"
        TF_PLAN_FILE          = "tfplan-${params.ENVIRONMENT}"
    }

    stages {

        // ── Checkout ──────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "Running pipeline for environment: ${params.ENVIRONMENT}"
            }
        }

        // ── Fix State Dir Permissions ─────────────────────────
        // Ensures Jenkins can read/write the local Terraform state
        // directory. This resolves "permission denied" errors on
        // terraform.tfstate.d/<env>/terraform.tfstate.backup
        stage('Fix Permissions') {
            steps {
                sh """
                   sudo mkdir -p terraform.tfstate.d/${params.ENVIRONMENT}
                   sudo chmod -R 777 terraform.tfstate.d/
                    echo "✅ State directory permissions fixed"
                """
            }
        }

        // ── Terraform Init ────────────────────────────────────
        stage('Terraform Init') {
            steps {
                sh """
                    echo "==> Terraform Init [ ${params.ENVIRONMENT} ]"
                    terraform init -reconfigure -input=false
                    terraform workspace select ${params.ENVIRONMENT} || \
                        terraform workspace new ${params.ENVIRONMENT}
                    echo "Active workspace: \$(terraform workspace show)"
                """
            }
        }

        // ── Terraform Plan ────────────────────────────────────
        stage('Terraform Plan') {
            steps {
                sh """
                    echo "==> Terraform Plan [ ${params.ENVIRONMENT} ]"
                    terraform plan \
                        -var-file="${env.TF_VAR_FILE}" \
                        -out="${env.TF_PLAN_FILE}" \
                        -input=false
                """
            }
            post {
                always {
                    sh """
                        terraform show -no-color "${env.TF_PLAN_FILE}" \
                            > "${env.TF_PLAN_FILE}.txt" 2>/dev/null || true
                    """
                    archiveArtifacts artifacts: "tfplan-${params.ENVIRONMENT}.txt",
                                     allowEmptyArchive: true
                }
            }
        }

        // ── Approval Gate ─────────────────────────────────────
        stage('Approval') {
            steps {
                script {
                    def msg = params.ENVIRONMENT == 'prod'
                        ? "PRODUCTION deployment — are you sure?"
                        : "Approve Apply for ${params.ENVIRONMENT.toUpperCase()}?"

                    input message: msg, ok: "Yes, Apply ${params.ENVIRONMENT.toUpperCase()}"
                }
            }
        }

        // ── Terraform Apply ───────────────────────────────────
        stage('Terraform Apply') {
            steps {
                sh """
                    echo "==> Terraform Apply [ ${params.ENVIRONMENT} ]"
                    terraform apply -input=false -auto-approve "${env.TF_PLAN_FILE}"
                """
            }
        }

        // ── Show Outputs ──────────────────────────────────────
        stage('Terraform Output') {
            steps {
                sh """
                    echo "==> Outputs for ${params.ENVIRONMENT}"
                    terraform output || true
                """
            }
        }

    }

    post {
        always {
            sh "rm -f ${env.TF_PLAN_FILE} ${env.TF_PLAN_FILE}.txt || true"
        }
        success {
            echo "✅ ${params.ENVIRONMENT.toUpperCase()} deployed successfully!"
        }
        failure {
            echo "❌ Pipeline failed for ${params.ENVIRONMENT.toUpperCase()} — check logs."
        }
    }

}
