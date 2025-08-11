## What is Jenkins?
---
Jenkins is an **automation server** used to build, test, and deploy software.

## What can Jenkins do?
---
Jenkins can **automate** repetitive tasks when developing software such as,
- Pulling latest code from remote repository
- Building and compiling code
- Running tests
- Packaging application
- Deploying to server

### How does Jenkins do it?
---
User can set up a **pipeline** to define the work that should be done on Jenkins using a file called Jenkinsfile.

#### Jenkinsfile example:
```groovy
pipeline {
    agent {
        docker {
            image 'mcr.microsoft.com/dotnet/sdk:9.0'
        }
    }

    environment {
        DOTNET_CLI_HOME = "${env.WORKSPACE}/.dotnet"
        RUNTIME_ID      = 'linux-arm64'
        PROJECT_PATH    = 'WebApplication1.csproj'
        PUBLISH_DIR     = 'publish'
        PACKAGE_DIR     = 'build/package'
        BIN_DIR         = "${PACKAGE_DIR}/usr/lib/weather-app"
        SERVICE_DIR     = "${PACKAGE_DIR}/usr/lib/systemd/system"
        DEBIAN_DIR      = "${PACKAGE_DIR}/DEBIAN"
        LAUNCHER_PATH   = "${PACKAGE_DIR}/usr/bin/weather-app"
        PACKAGE_NAME    = 'weather-app'
        VERSION         = '1.0.0'
        SERVICE_NAME    = 'weather-app.service'
        OUTPUT_DEB      = "${PACKAGE_NAME}_${VERSION}_arm64.deb"
    }

    stages {
        stage('Clean') {
            steps {
                sh """
                    rm -rf ${PUBLISH_DIR} build obj bin
                    mkdir -p ${BIN_DIR}
                    mkdir -p ${SERVICE_DIR}
                    mkdir -p ${DEBIAN_DIR}
                    mkdir -p ${PACKAGE_DIR}/usr/bin
                """
            }
        }

        stage('Restore Dependencies') {
            steps {
                sh "dotnet restore ${PROJECT_PATH}"
            }
        }

        stage('Publish .NET App') {
            steps {
                sh """
                    dotnet publish ${PROJECT_PATH} \\
                        -c Release \\
                        -r ${RUNTIME_ID} \\
                        --self-contained true \\
                        -o ${PUBLISH_DIR}
                """
            }
        }

        stage('Copy Files & Write Metadata') {
            steps {
                // Copy application files
                sh "cp -r ${PUBLISH_DIR}/* ${BIN_DIR}/"

                // Control file
                writeFile file: "${DEBIAN_DIR}/control", text: """
                    Package: ${PACKAGE_NAME}
                    Version: ${VERSION}
                    Section: web
                    Priority: optional
                    Architecture: arm64
                    Maintainer: Your Name <your@email.com>
                    Description: Weather App
                """.stripIndent()

                // postinst
                writeFile file: "${DEBIAN_DIR}/postinst", text: """
                    #!/bin/bash
                    set -e
                    systemctl daemon-reexec
                    systemctl daemon-reload
                    systemctl enable ${SERVICE_NAME}
                    systemctl start ${SERVICE_NAME}
                """.stripIndent()
                sh "chmod 755 ${DEBIAN_DIR}/postinst"

                // prerm
                writeFile file: "${DEBIAN_DIR}/prerm", text: """
                    #!/bin/bash
                    set -e
                    systemctl stop ${SERVICE_NAME} || true
                    systemctl disable ${SERVICE_NAME} || true
                """.stripIndent()
                sh "chmod 755 ${DEBIAN_DIR}/prerm"

                // systemd service
                writeFile file: "${SERVICE_DIR}/${SERVICE_NAME}", text: """
                    [Unit]
                    Description=Weather App Service
                    After=network.target

                    [Service]
                    Type=simple
                    ExecStart=/usr/lib/weather-app/WebApplication1
                    Restart=always
                    RestartSec=5
                    Environment=DOTNET_ENVIRONMENT=Production
                    StandardOutput=journal
                    StandardError=journal

                    [Install]
                    WantedBy=multi-user.target
                """.stripIndent()

                // launcher script
                writeFile file: "${LAUNCHER_PATH}", text: """
                    #!/bin/bash
                    sudo /usr/lib/weather-app/WebApplication1
                """.stripIndent()
                sh "chmod 755 ${LAUNCHER_PATH}"
            }
        }

        stage('Build .deb Package') {
            steps {
                sh "dpkg-deb --build ${PACKAGE_DIR} ${OUTPUT_DEB}"
            }
        }

        stage('Archive .deb') {
            steps {
                archiveArtifacts artifacts: "${OUTPUT_DEB}", fingerprint: true
            }
        }
    }

    post {
        always {
            sh "rm -rf ${PUBLISH_DIR} build obj bin"
        }
        failure {
            sh """
                echo "=== Debug Information ==="
                dotnet --version
                echo "Project files:"
                ls -la *.csproj || echo "No .csproj files found"
                echo "NuGet sources:"
                dotnet nuget list source
                echo "Current directory contents:"
                ls -la
                echo "obj directory (if exists):"
                ls -la obj/ || echo "No obj directory"
            """
        }
    }
}

```