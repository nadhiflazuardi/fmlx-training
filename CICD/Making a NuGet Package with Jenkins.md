## Side note
---
- For efficiency, make sure everything works locally first before working on the server
- Local directory can also be set as local nuget repository

## Example Structure (Project + Unit Test)
```
HelloWorldSolution.sln
HelloWorldClassLib/
  HelloWorldClassLib.csproj
HelloWorldClassLib.Test/
  HelloWorldClassLib.Test.csproj
Jenkinsfile

```

## Jenkinsfile Example
```groovy
pipeline {
    agent {
        docker {
            image 'mcr.microsoft.com/dotnet/sdk:9.0'
        }
    }

    environment {
        DOTNET_CLI_HOME = "${env.WORKSPACE}/.dotnet"
        PROJECT_PATH = 'HelloWorldClassLib/HelloWorldClassLib.csproj'
        TEST_PROJECT_PATH = 'HelloWorldClassLib.Test/HelloWorldClassLib.Test.csproj'
        PACKAGE_OUTPUT_DIR = 'nupkg'
    }

    stages {
        stage('Clean') {
            steps {
                sh """
                    rm -rf bin obj ${PACKAGE_OUTPUT_DIR}
                    mkdir -p ${PACKAGE_OUTPUT_DIR}
                """
            }
        }

        stage('Restore') {
            steps {
                sh "dotnet restore"
            }
        }

        stage('Build') {
            steps {
                sh "dotnet build ${PROJECT_PATH} --configuration Release"
            }
        }

        stage('Test') {
            steps {
                sh "dotnet test ${TEST_PROJECT_PATH} --no-build"
            }
        }

        stage('Pack NuGet') {
            steps {
                sh """
                    dotnet pack ${PROJECT_PATH} \
                        --configuration Release \
                        --output ${PACKAGE_OUTPUT_DIR}
                """
            }
        }

        stage('Archive .nupkg') {
            steps {
                archiveArtifacts artifacts: "${PACKAGE_OUTPUT_DIR}/*.nupkg", fingerprint: true
            }
        }
    }

    post {
        always {
            sh "rm -rf bin obj"
        }
        failure {
            sh """
                echo "Build failed. Debug info:"
                dotnet --version
                dotnet nuget list source
                echo "Directory contents:"
                ls -R
            """
        }
    }
}

```

