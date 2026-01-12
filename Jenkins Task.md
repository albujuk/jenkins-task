# Jenkins-Task

## Setup Jenkins
We want to install Jenkins on Ubuntu as BareMetal.
```sh
sudo apt update
sudo apt install fontconfig openjdk-21-jre

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins

sudo systemctl enable --now jenkins
```

After Setting up the accounts and install the recommended plugins including the "Docker Pipeline" Plugin, we add the creds for Docker hub and GitHub accounts in the "Credentials" Page from "Manage Jenkins".

![[img_01.png]]

## 1. Build Docker Image from a Private GitHub Repo
```groovy
node {
    def app
}
pipeline {
    agent any
    parameters {
        choice(name: 'Branch', choices: ['main', 'dev', 'test'], description: 'Choose a Branch')
        string(name: 'Tag', defaultValue: '', description: 'Specify the image tag.')
    }
    stages {
        stage('Fetch Code') {
            steps{
                git branch: "${params.Branch}", credentialsId: 'gh-creds', url: 'https://github.com/albujuk/ifconfig.py'
                script{
                    env.CommitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
                echo "Commit Hash is XYZ${CommitHash}XYZ"
            }
        }
        stage('Build Image'){
            steps{
                script{
                    env.ImageTag = params.Tag?: env.CommitHash
                    
                    app = docker.build("albujuk/ifconfig.py:${ImageTag}")   
                }
                echo "Done Building the Image ${app.imageName()}"
            }
        }
        stage('Push Image'){
            steps{
                script{
                    docker.withRegistry('', 'docker-creds'){
                        app.push()            
                    }
                }
            }
        }
    }
}
```

This is a Jenkins pipeline that pulls a code from a private GitHub repo in the first stage then it builds a docker image from a Docker file located in the repo, then it pushes the image to a private docker hub registry.

![[Pasted image 20260112222657.png]]

> [!Warning] Disclaimer
> The Branches are Static in this example and I believe there is a way to do it dynamically.


## 2. Deploy the app on the server
```groovy
node {
    stage('run'){
        docker.withRegistry('', 'docker-creds'){
            docker.image('albujuk/ifconfig.py').run()
        }     
    }
}
```

This is a Jenkins pipeline that runs a container from the image that we built and pushed to a private docker hub repo earlier, it uses the credentials we created to login since the image is hosted privately.

## 3. Cronjob pipeline
```groovy
node {
    stage('check')
    {
        sh '''
        if ((docker images -f "dangling=true" -q | wc -l) > 10); then
            docker image prune
        fi
        '''
    }     
}
```

This pipeline is more about bash rather than it's an actual pipeline, and for the cron part we do it as the following.
![[img_02.png]]

---
# Conclusion 
We built 3 pipelines and gained Jenkins experience across various topics.

![[img_03.png]]
