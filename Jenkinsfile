pipeline {

    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'uat', 'prod'],
            description: 'Select environment'
        )
        choice(
            name: 'ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Select Terraform action'
        )
    }

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        TF_VAR_FILE  = "envs/${params.ENVIRONMENT}.tfvars"
        TF_PLAN_FILE = "tfplan-${params.ENVIRONMENT}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Env: ${params.ENVIRONMENT}, Action: ${params.ACTION}"
            }
        }

        // ❌ REMOVE this if using S3 backend (recommended)
        // Keeping it safe (no sudo)
        stage('Fix Permissions') {
            when {
                expression { return false }   // disabled
            }
            steps {
                sh """
                    mkdir -p terraform.tfstate.d/${params.ENVIRONMENT}
                    chmod -R 777 terraform.tfstate.d/
                """
            }
        }

        stage('Terraform Init') {
            steps {
                sh """
                    terraform init -reconfigure -input=false
                    terraform workspace select ${params.ENVIRONMENT} || \
                    terraform workspace new ${params.ENVIRONMENT}
                """
            }
        }

        // ── PLAN ─────────────────────────────
        stage('Terraform Plan') {
            when {
                expression { params.ACTION == 'plan' || params.ACTION == 'apply' }
            }
            steps {
                sh """
                    terraform plan \
                    -var-file="${env.TF_VAR_FILE}" \
                    -out="${env.TF_PLAN_FILE}" \
                    -input=false
                """
            }
        }

        // ── APPROVAL ─────────────────────────
        stage('Approval') {
            when {
                expression { params.ACTION == 'apply' || params.ACTION == 'destroy' }
            }
            steps {
                script {
                    def msg = (params.ACTION == 'destroy') ?
                        "⚠️ DESTROY ${params.ENVIRONMENT.toUpperCase()} — Are you sure?" :
                        "Approve APPLY for ${params.ENVIRONMENT.toUpperCase()}?"

                    input message: msg, ok: "Proceed"
                }
            }
        }

        // ── APPLY ────────────────────────────
        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh """
                    terraform apply -input=false -auto-approve "${env.TF_PLAN_FILE}"
                """
            }
        }

        // ── DESTROY ──────────────────────────
        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                sh """
                    terraform destroy \
                    -var-file="${env.TF_VAR_FILE}" \
                    -auto-approve
                """
            }
        }

        // ── OUTPUT ───────────────────────────
        stage('Terraform Output') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh "terraform output || true"
            }
        }
    }

    post {
        always {
            sh "rm -f ${env.TF_PLAN_FILE} ${env.TF_PLAN_FILE}.txt || true"
        }
        success {
            echo "✅ ${params.ACTION.toUpperCase()} SUCCESS for ${params.ENVIRONMENT}"
        }
        failure {
            echo "❌ ${params.ACTION.toUpperCase()} FAILED for ${params.ENVIRONMENT}"
        }
    }
}