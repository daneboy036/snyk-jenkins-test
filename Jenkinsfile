pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        TF_IN_AUTOMATION   = 'true'
        SNYK_ORG = credentials('snyk-org-slug')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Check if we have authoriazation to use snyk, snyk-linux is what it's called in the path
        // even if snyk-linux iac fails, we'll keep going ( || true)
        // we do this because the monitor will use the actual plugin instead of the cli within the plugin
        // we have to set the directory because like jenkins, the snyk plugin will run in the root of the repo, but our terraform code is in week29/terraform
        stage('Snyk IaC Scan Test') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token-string', variable: 'SNYK_TOKEN')]) {
                    sh '''
                        export PATH=$PATH:/var/lib/jenkins/tools/io.snyk.jenkins.tools.SnykInstallation/snyk
                        snyk-linux auth $SNYK_TOKEN
                        snyk-linux iac test --org=$SNYK_ORG --severity-threshold=high || true
                    '''
                }
            }
        }


        
        stage('Snyk IaC Scan Monitor') {
            steps {
                snykSecurity(
                    snykInstallation: 'snyk',
                    snykTokenId: 'snyk-api-token',
                    additionalArguments: '--iac --report --org=$SNYK_ORG --severity-threshold=high --debug',
                    failOnIssues: true,
                    monitorProjectOnBuild: false
                )
                }
        }
        
        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Terraform Plan') {
            steps {
                sh 'terraform plan -out=tfplan' 
            }
        }

        stage('Terraform Apply') {
            steps {
                sh 'terraform apply -auto-approve tfplan'
            }
        }

        stage('Optional Destroy') {
            steps {
                script {
                    def destroyChoice = input(
                        message: 'Do you want to run terraform destroy?',
                        ok: 'Submit',
                        parameters: [
                            choice(
                                name: 'DESTROY',
                                choices: ['no', 'yes'],
                                description: 'Select yes to destroy resources'
                            )
                        ]
                    )
                    if (destroyChoice == 'yes') {
                        sh 'terraform destroy -auto-approve'
                    } else {
                        echo "Skipping destroy"
                    }
                }
            }
        }
    }
}