pipeline {
  agent any

  environment {
    // Name of the Jenkins credential (SSH username + private key)
    SSH_CREDENTIALS_ID = 'ec2-ssh-key'      // create this in Jenkins credentials (see steps)
    // SSH user used by the AMI (change to "ubuntu" if using Ubuntu AMIs)
    SSH_USER = 'ec2-user'
    // The list of target instance IPs / hostnames (we will set them as pipeline parameters)
    // You can also hardcode or pass via environment variable; using parameters is flexible.
  }

  parameters {
    string(name: 'TARGETS', defaultValue: '44.201.228.249,52.91.219.245', description: 'Comma-separated target IPs or hostnames (public IPs of EC2 web1, web2)')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Build / Prepare') {
      steps {
        // If the repo needs a build step add here (npm build/mvn etc). Kelownatrails is likely static/site - we just deploy.
        echo 'Prepare artifacts to deploy...'
      }
    }

    stage('Deploy to Production') {
      steps {
        script {
          // normalize parameter into list
          def targets = params.TARGETS.split(',').collect { it.trim() }
          // Use Jenkins SSH credential and scp/ssh inside ssh-agent
          sshagent([env.SSH_CREDENTIALS_ID]) {
            // Create a tar of repo contents (or pick the directory you need)
            sh 'tar -czf site_package.tar.gz -C . $(git ls-files) || tar -czf site_package.tar.gz .'

            for (t in targets) {
              echo "Deploying to ${t}..."
              // Copy artifact
              sh "scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null site_package.tar.gz ${env.SSH_USER}@${t}:/tmp/site_package.tar.gz"
              // Run remote install & deploy commands (assumes instance has sudo and ssh user can sudo)
              sh """ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${env.SSH_USER}@${t} << 'EOF'
                set -e
                mkdir -p /var/www/kelownatrails
                tar -xzf /tmp/site_package.tar.gz -C /var/www/kelownatrails
                # Basic install: ensure nginx is installed and running (works for Amazon Linux 2)
                if ! command -v nginx >/dev/null 2>&1; then
                  sudo yum update -y || true
                  sudo yum install -y nginx || sudo apt-get update -y && sudo apt-get install -y nginx
                fi
                sudo systemctl enable nginx || true
                sudo systemctl start nginx
                # point nginx default root to our site (simple config replacement)
                sudo bash -c 'cat > /etc/nginx/conf.d/kelownatrails.conf <<NGINX
server {
  listen 80;
  server_name _;
  root /var/www/kelownatrails;
  index index.html;
  location / {
    try_files \$uri \$uri/ =404;
  }
}
NGINX'
                sudo nginx -s reload || sudo systemctl restart nginx
                echo "Deployed to ${t}"
              EOF
              """
            }
          } // sshagent
        }
      }
    }

    stage('Smoke Test') {
      steps {
        script {
          // We can curl the ALB DNS name if provided as parameter or the first target
          echo "Running quick curl against the first target..."
          def firstTarget = params.TARGETS.split(',')[0].trim()
          sh "curl -I http://${firstTarget} || true"
        }
      }
    }
  } // stages

  post {
    always {
      archiveArtifacts artifacts: 'site_package.tar.gz', allowEmptyArchive: true
    }
    success {
      echo 'Deployment finished successfully.'
    }
    failure {
      echo 'Deployment failed — check console output.'
    }
  }
}