pipeline {
  agent any

  environment {
    SSH_CREDENTIALS_ID = 'ec2-ssh-key'   // must match the Jenkins credential ID you created
    // default SSH user (used only if credential username is missing) - optional
    DEFAULT_SSH_USER = 'ec2-user'
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
        echo 'Prepare artifacts to deploy...'
      }
    }

    stage('Deploy to Production') {
      steps {
        script {
          def targets = params.TARGETS.split(',').collect { it.trim() }.findAll { it }

          // Use credentials binding to get a temporary key file and username
          withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CREDENTIALS_ID,
                                            keyFileVariable: 'SSH_KEYFILE',
                                            usernameVariable: 'SSH_USER_VAR')]) {

            // Ensure we have a username (fallback to DEFAULT_SSH_USER)
            def sshUser = (env.SSH_USER_VAR?.trim()) ? env.SSH_USER_VAR.trim() : env.DEFAULT_SSH_USER
            echo "Using SSH user: ${sshUser}"
            echo "Temporary SSH key file: ${SSH_KEYFILE}"

            // Create artifact: prefer tracked git files, fallback to whole workspace
            sh '''
              if git rev-parse --git-dir >/dev/null 2>&1; then
                FILES=$(git ls-files)
                if [ -n "$FILES" ]; then
                  tar -czf site_package.tar.gz $FILES
                else
                  tar -czf site_package.tar.gz .
                fi
              else
                tar -czf site_package.tar.gz .
              fi
            '''

            for (t in targets) {
              echo "Deploying to ${t}..."
              // copy artifact (use -i with the temporary key file)
              sh "scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i ${SSH_KEYFILE} site_package.tar.gz ${sshUser}@${t}:/tmp/site_package.tar.gz"

              // execute remote deployment commands
              sh """ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i ${SSH_KEYFILE} ${sshUser}@${t} <<'EOF'
set -e
# create web root and extract
sudo mkdir -p /var/www/kelownatrails
sudo tar -xzf /tmp/site_package.tar.gz -C /var/www/kelownatrails --overwrite
# install nginx if missing (supports yum or apt)
if ! command -v nginx >/dev/null 2>&1; then
  if command -v yum >/dev/null 2>&1; then
    sudo yum update -y || true
    sudo yum install -y nginx
  elif command -v apt-get >/dev/null 2>&1; then
    sudo apt-get update -y
    sudo apt-get install -y nginx
  fi
fi
sudo systemctl enable nginx || true
sudo systemctl start nginx || true
# place simple nginx config for kelownatrails
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
sudo nginx -s reload || sudo systemctl restart nginx || true
echo "Deployed to ${t}"
EOF
"""
            }
          } // withCredentials
        } // script
      } // steps
    } // stage

    stage('Smoke Test') {
      steps {
        script {
          def firstTarget = params.TARGETS.split(',')[0].trim()
          echo "Running quick curl against ${firstTarget}..."
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
/*pipeline {
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
// inside your script block, replacing sshagent([...]) { ... }



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
} */