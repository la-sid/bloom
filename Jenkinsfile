// Jenkins pipeline (Windows) for a static website. Needs only Node.js on the Jenkins machine.
pipeline {
    agent any

    environment {
        APP_DIR  = 'C:\\mywebsite'   // where the live copy of the site is served from
        APP_PORT = '3002'
    }

    options {
        timestamps()
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Check Files') {
            steps {
                bat 'node -v'
                // Fail early if the homepage is missing
                bat 'if not exist index.html (echo index.html is missing & exit /b 1)'
            }
        }

        stage('Deploy') {
            steps {
                // 1. Stop the previous version if it is running
                bat returnStatus: true, script: '''powershell -NoProfile -Command "Get-NetTCPConnection -LocalPort %APP_PORT% -State Listen -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }"'''

                // 2. Copy the site (skipping test and tooling folders). robocopy codes below 8 mean success.
                script {
                    def rc = bat(returnStatus: true, script: 'robocopy "%WORKSPACE%" "%APP_DIR%" /E /XD .git .vscode node_modules test-results playwright-report playwright-test /XF Jenkinsfile .gitignore /NFL /NDL /NJH /NJS /NP')
                    if (rc >= 8) { error("Copy failed (robocopy code ${rc})") }
                }

                // 3. Start the web server in the background so it keeps running after the build ends
                withEnv(['BUILD_ID=dontKillMe', 'JENKINS_NODE_COOKIE=dontKillMe']) {
                    bat '''powershell -NoProfile -Command "Start-Process -FilePath (Get-Command node).Source -ArgumentList 'serve.js' -WorkingDirectory '%APP_DIR%' -WindowStyle Hidden -RedirectStandardOutput '%TEMP%\\mywebsite.log' -RedirectStandardError '%TEMP%\\mywebsite-error.log'"'''
                }
            }
        }

        stage('Verify') {
            steps {
                // Fails the build unless the homepage is really being served
                bat '''powershell -NoProfile -Command "$ok=$false; for($i=0;$i -lt 20;$i++){ try { $r=Invoke-WebRequest -UseBasicParsing http://localhost:%APP_PORT%/; if($r.StatusCode -eq 200){ $ok=$true; Write-Output 'Website is live'; break } } catch {}; Start-Sleep 1 }; if(-not $ok){ exit 1 }"'''
            }
        }
    }

    post {
        success { echo "Done! Open http://localhost:${env.APP_PORT} in your browser." }
        failure { echo 'Pipeline failed. Open the failed stage in Console Output.' }
    }
}
