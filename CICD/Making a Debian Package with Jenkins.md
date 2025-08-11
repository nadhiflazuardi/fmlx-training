## Side Note
---
- Everything alongside the `DEBIAN/` folder inside package directory becomes the file structure that will be created on the target system

## Minimal Layout of Deb Package
myapp/
├── DEBIAN/
│   └── control
└── usr/
    └── bin/
        └── myapp

1. DEBIAN/control file
	Contains package metadata. Minimal example:
	```bash
	Package: myapp
	Version: 1.0
	Architecture: all
	Maintainer: Your Name <you@example.com>
	Description: A minimal Debian package
```

2. usr/bin/myapp (the app itself)
	Can be binary or script. Simple example:
```
	#!/bin/bash
	echo "Hello from myapp"
```

## (Optional) Maintainer Scripts
---
Inside DEBIAN folder, alongside control file that contains metadata for the .deb package, it's common to put executable shell scripts with different purposes
### preinst
- When it runs: before files are unpacked
- Typical use: stop a running service, create users, backup configs
- Example:
```bash
#!/bin/bash
set -e

echo "[preinst] Preparing for installation..."

# Stop the service if it's running (upgrade or reinstall case)
if systemctl is-active --quiet weather-app.service; then
    echo "[preinst] Stopping weather-app service..."
    systemctl stop weather-app.service
fi
```

### postinst
- When it runs: After files are unpacked
- Typical use: Start services, reload daemons
- Example:
```bash
#!/bin/bash
set -e

echo "[postinst] Finalizing installation..."

systemctl daemon-reexec
systemctl daemon-reload
systemctl enable weather-app.service
systemctl start weather-app.service
```

### prerm
- When it runs: Before package removal
- Typical use: stop services, clean up temp files
- Example:
```bash
#!/bin/bash
set -e

echo "[prerm] Preparing to remove weather-app..."

systemctl stop weather-app.service || true
systemctl disable weather-app.service || true
```

### postrm
- When it runs: After package removal
- Typical use: Remove configs, users, or leftover files
- Example:
```bash
#!/bin/bash
set -e

echo "[postrm] Cleaning up after removal..."

# Remove leftover data directory (optional)
if [ -d /var/lib/weather-app ]; then
    rm -rf /var/lib/weather-app
    echo "[postrm] Removed /var/lib/weather-app"
fi
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
        RUNTIME_ID = 'linux-arm64'
        PROJECT_PATH = 'WebApplication1.csproj'
        PUBLISH_DIR = 'publish'
        PACKAGE_DIR = 'build/package'
        BIN_DIR = "${PACKAGE_DIR}/usr/lib/weather-app"
        SERVICE_DIR = "${PACKAGE_DIR}/usr/lib/systemd/system"
        DEBIAN_DIR = "${PACKAGE_DIR}/DEBIAN"
        LAUNCHER_PATH = "${PACKAGE_DIR}/usr/bin/weather-app"
        PACKAGE_NAME = 'weather-app'
        VERSION = '1.0.0'
        SERVICE_NAME = 'weather-app.service'
        OUTPUT_DEB = "${PACKAGE_NAME}_${VERSION}_arm64.deb"
    }

    stages {
        stage('Clean') {
            steps {
                sh """
                    rm -rf ${PUBLISH_DIR} build obj bin
                    mkdir -p ${BIN_DIR} ${SERVICE_DIR} ${DEBIAN_DIR} ${PACKAGE_DIR}/usr/bin
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
                    dotnet publish ${PROJECT_PATH} \
                        -c Release \
                        -r ${RUNTIME_ID} \
                        --self-contained true \
                        -o ${PUBLISH_DIR}
                """
            }
        }

        stage('Copy Files & Write Metadata') {
            steps {
                // Copy app
                sh "cp -r ${PUBLISH_DIR}/* ${BIN_DIR}/"

                // DEBIAN/control
                writeFile file: "${DEBIAN_DIR}/control", text: """
                    Package: ${PACKAGE_NAME}
                    Version: ${VERSION}
                    Section: web
                    Priority: optional
                    Architecture: arm64
                    Maintainer: Your Name <your@email.com>
                    Description: Weather App
                """.stripIndent()

                // postinst script
                writeFile file: "${DEBIAN_DIR}/postinst", text: """
                    #!/bin/bash
                    set -e
                    systemctl daemon-reexec
                    systemctl daemon-reload
                    systemctl enable ${SERVICE_NAME}
                    systemctl start ${SERVICE_NAME}
                """.stripIndent()
                sh "chmod 755 ${DEBIAN_DIR}/postinst"

                // prerm script
                writeFile file: "${DEBIAN_DIR}/prerm", text: """
                    #!/bin/bash
                    set -e
                    systemctl stop ${SERVICE_NAME} || true
                    systemctl disable ${SERVICE_NAME} || true
                """.stripIndent()
                sh "chmod 755 ${DEBIAN_DIR}/prerm"

                // systemd service file
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
                ls -la *.csproj || echo "No .csproj found"
                dotnet nuget list source
                ls -la
                ls -la obj/ || echo "No obj directory"
            """
        }
    }
}

```