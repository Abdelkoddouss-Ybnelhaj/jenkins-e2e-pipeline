# Jenkins CI/CD Pipeline Documentation

This repository contains a full Jenkins CI/CD pipeline implementation for a monorepo project using **React** (frontend) and **.NET Core** (backend), with stages for building, testing, SonarQube analysis, Docker image creation, and deployment using **Docker Compose**.


## 🧾 Table of Contents

- [🧩 Architecture Overview](#architecture-overview)
- [📦 Stack & Tools](#stack--tools)
- [📁 Project Structure](#project-structure)
- [⚙️ Prerequisites](#prerequisites)
- [🐳 Docker Compose Setup](#Docker-Compose-Setup)
- [🔧 Jenkins Pipeline Stages](#jenkins-pipeline-stages)
- [📝 Jenkinsfile Snippets](#jenkinsfile-snippets)
- [🛠 Configuration Details](#configuration-details)
- [🐞 Testing & Quality](#testing--quality)
- [🔐 Credentials & Secrets](#credentials--secrets)
- [🚀 Deployment](#deployment)
- [🧪 Troubleshooting](#troubleshooting)
- [🔗 References](#references)

## Architecture

![Alt test](./assets/Jenkins-CICD%20(1).png)

## 📦 Stack & Tools

| Tool        | Version      | Purpose                          |
|-------------|--------------|----------------------------------|
| Jenkins     | 2.426+       | CI/CD automation                 |
| SonarQube   | 10.x         | Code quality analysis            |
| Docker      | 24.x         | Containerization                 |
| GitLab      | Cloud-hosted | SCM and pipeline trigger         |
| .NET SDK    | 8.x          | Backend build and test           |
| Node.js     | 18.x         | React frontend build/test        |
| Docker Compose | 3.x     | Orchestration of local services  |



## ⚙️ Prerequisites
- Jenkins (with required plugins)

- GitLab repository access

- Docker installed on Jenkins agent

- SonarQube server + scanner

- JDK (for backend analysis)

## 🐳 Docker Compose Setup

- checkout the [docker-compose.yml](./docker-compose.yml)




## 🔌 Required Jenkins Plugins
- GitLab Plugin

- Pipeline

- Docker Pipeline

- SonarQube Scanner

- JUnit

- Performance Plugin (for test metrics)

- Jira Plugin


## 🔄 Jenkins Pipeline Stages

| Stage                  | Description                                        |
| ---------------------- | -------------------------------------------------- |
| **Checkout**           | Pulls the code from the GitLab repository          |
| **Build**              | Builds the React app and .NET backend              |
| **Test**               | Runs unit tests                  |
| **SonarQube Analysis** | Performs static code analysis                      |
| **Docker Build**       | Builds Docker images for both frontend and backend |
| **Deployment**         | Deploys the containers to the **test environment** |




# 1. **🔍 Checkout**


**STEP 1: Generate a gitlab access token**

- Go to gitlab > Settings > Access Tokens
- Give the token a name (ex: `jenkins-token`)
- give it permissions (ex: `api` , `read_repository` , `write_repository`)


**STEP 2: Configure gitlab credentials in jenkins**

- Go to manage jenkins > credentials > add credentials
    - kind: Username with Password 
    - Name: GitLab
    - Password: the PAT from step 1

**STEP 3: Create jenkins job**
- Specify triggers
- Code snippet
```bash
stage('Checkout') {
            steps {
                git url: 'https://gitlab.com/owner/your-project.git',
                    credentialsId: 'your-cred-id',
                    branch: 'master'
            }

```

**STEP 3: configure gitlab webhook**
- Go to your Gitlab repo > settings > Webhooks
    - URL: http://<jenkins-url>/project/<job-name>
    - specify triggers



--
# 3. **SonarQube Analysis**

**> Communication Flow**

- Jenkins builds your project

- Jenkins uses the dotnet-sonarscanner to push analysis to SonarQube

- SonarQube processes the report

- SonarQube sends the result back to Jenkins via the webhook

- Jenkins receives it and the pipeline continues

**> 3.1 Create a token in SonarQube**

- Go to your sonarqube server
- Select the Profile > MyAccount
- Security > generate token


**> 3.2 Configure Jenkins to Talk to SonarQube**

- Go to Manage Jenkins → Configure System

- Scroll to SonarQube servers

- Click Add SonarQube

- Name: sonarqube-server (you will reference it by this name)

- Server URL: e.g., http://localhost:9000 or http://sonarqube:9000 (if Dockerized)

- Authentication Token:

    - Create it from SonarQube UI:
My Account → Security → Generate Token

    - Store the token in Jenkins as a "Secret Text" credential

- Select the token in the "Server authentication token" field

✅ Tick "Environment variables" checkbox if available

**> 3.3 Setup SonarQube Webhook for Quality Gate**

For `waitForQualityGate` to work:

- Go to SonarQube → Administration → Configuration → Webhooks

- Add webhook:

    - Name: Jenkins

    - URL: If Jenkins runs as a Docker service: http://jenkins:8080/sonarqube-webhook/ Or your Jenkins public/host IP

Test the URL manually or from SonarQube logs to ensure it reaches Jenkins.


**> 3.4 Code Snippet**

```bash
stage('SonarQube Analysis') {
    steps 
    {
        withSonarQubeEnv('sonarqube-server') {
            sh '''
                dotnet tool install --global dotnet-sonarscanner || true
                export PATH="$PATH:$HOME/.dotnet/tools"
                dotnet sonarscanner begin /k:"weather-api"                        
                dotnet build
                dotnet sonarscanner end
                    '''
                }
    }
}

 stage("SonarQube Quality Gate Check") {
        steps 
            {
                script {
                def qualityGate = waitForQualityGate()
                    
                    if (qualityGate.status != 'OK') {
                        echo "${qualityGate.status}"
                        error "Quality Gate failed: ${qualityGateStatus}"
                    }
                    else {
                        echo "${qualityGate.status}"
                        echo "SonarQube Quality Gates Passed"
                    }
            }
}

```
--
# 4. **Sonarqube Metrics Definition**

**🛠️ Maintainability**

- `Code Smells:` Indicate maintainability issues that may not be bugs but could lead to problems in the future.

- `Maintainability Rating:` Grades the code from A (best) to E (worst) based on the number and severity of code smells.
Medium

- `Technical Debt:` Estimates the time required to fix all maintainability issues.

**🐞 Reliability**
- `Bugs:` Identify defects in the code that could lead to incorrect behavior.

- `Reliability Rating:` Grades the code from A to E based on the severity of bugs.

- `Reliability Remediation Effort:` Estimates the time needed to fix all reliability issues.
docs.sonarsource.com

**🔒 Security**

- `Vulnerabilities:` Highlight security-related issues that could be exploited.

- `Security Rating:` Grades the code from A to E based on the severity of vulnerabilities.

- `Security Remediation Effort:` Estimates the time required to address all security issues.

- `Security Hotspots:` Code sections that require manual review to ensure they are secure.
Wikipedia

- `Security Review Rating:` Grades the thoroughness of security hotspot reviews.
docs.sonarsource.com

**✅ Test Coverage**

- `Coverage:` Combines line and condition coverage to show the percentage of code tested.

- `Line Coverage:` Percentage of executable lines covered by tests.
docs.sonarsource.com

- `Condition Coverage:` Percentage of boolean expressions evaluated both to true and false during testing.
docs.sonarsource.com

- `Uncovered Lines/Conditions`: Lines or conditions not covered by tests.

**🧠 Complexity**

- `Cyclomatic Complexity:` Measures the number of independent paths through the code, indicating its complexity.
docs.sonarsource.com

- `Cognitive Complexity:` Assesses how difficult the code is to understand, considering control flow and nesting.
docs.sonarsource.com


**🔁 Duplications**

- `Duplicated Lines:` Number of lines that are duplicated in the codebase.

- `Duplicated Blocks:` Number of duplicated code blocks.

- `Duplicated Files:` Number of files containing duplicated code.

- `Duplicated Lines Density:` Percentage of the codebase that is duplicated.


**🚦 Quality Gates**

- `Quality Gate Status:` Indicates whether the project passes or fails the defined quality thresholds.
docs.sonarsource.com

- `Quality Gate Conditions:` Specific thresholds set for various metrics that determine the quality gate status.


--
## 5. **Email Notification**

**> 5.1 Install Email Notification plugin**
-  Manage Jenkins > Plugin Manager > Available.
- Search for Email Extension 


**> 5.2 Configure SMTP Server**
- Manage Jenkins > Configure System.
- E-mail Notification.
    - `SMTP server:`  smtp.gmail.com
    - `Default user e-mail suffix`: @gmail.com
    - `Use SMTP Authentication`:
        - User Name: youremail@gmail.com
        - Password: Go to https://myaccount.google.com/apppasswords
    - `Use TLS`
    
**> 5.3 Extract the author email**
```bash
def authorEmail = sh(script: "git log -1 --pretty=format:'%ae'", returnStdout: true).trim()
```
**> 5.4 Code Snippet**
```bash
script 
{
    def authorEmail = sh(script: "git log -1 --pretty=format:'%ae'", returnStdout: true).trim()
    emailext(
        subject: "Jenkins Build Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: buildFailureMessage(FAILURE_REASON),
        to: authorEmail,
        from: 'Jenkins <noreply@gmail.com>',
        mimeType: 'text/plain'
    )
}
