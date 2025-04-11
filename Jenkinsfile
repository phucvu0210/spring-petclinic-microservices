pipeline {
    agent any

    environment {
        MOD_FILES = ''
    }

    stages {
        stage('Detect Changes') {
            agent any
            steps {
                script {
                    def services = []
                    MOD_FILES = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim()
                    echo "🔍 Modified files: ${MOD_FILES}"

                    MOD_FILES.split("\n").each { file ->
                        if (file.startsWith("spring-petclinic-") && file.split("/").size() > 1) {
                            def svc = file.split("/")[0]
                            if (!services.contains(svc)) {
                                services << svc
                            }
                        }
                    }

                    if (services.isEmpty()) {
                        echo "✅ No changes detected, skipping."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    echo "⚙️ Affected services: ${services}"
                    env.SERVICES = services.join(',')
                }
            }
        }

        stage('Test & Coverage') {
            agent any
            when {
                expression { return env.SERVICES != null && env.SERVICES != "" }
            }
            steps {
                script {
                    def services = env.SERVICES.split(',')
                    services.each { svc ->
                        echo "🧪 Testing: ${svc}"
                        dir(svc) {
                            sh '../mvnw clean verify -PbuildDocker jacoco:report'
                            def jacocoFile = sh(script: "find target -name jacoco.xml", returnStdout: true).trim()

                            if (!jacocoFile) {
                                echo "⚠️ No JaCoCo report found for ${svc}."
                            } else {
                                def missed = sh(
                                    script: """awk -F'missed="' '/<counter type="LINE"/ {gsub(/".*/, "", \$2); sum += \$2} END {print sum}' ${jacocoFile}""",
                                    returnStdout: true
                                ).trim()

                                def covered = sh(
                                    script: """awk -F'covered="' '/<counter type="LINE"/ {gsub(/".*/, "", \$2); sum += \$2} END {print sum}' ${jacocoFile}""",
                                    returnStdout: true
                                ).trim()

                                def total = missed.toInteger() + covered.toInteger()
                                def coveragePercent = (total > 0) ? (covered.toInteger() * 100 / total) : 0

                                echo "🚀 Test coverage for ${svc}: ${coveragePercent}%"

                                if (coveragePercent < 70) {
                                    error("❌ Test coverage below 70% for ${svc}.")
                                }
                            }
                        }
                    }
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    script {
                        def services = env.SERVICES.split(',')
                        services.each { svc ->
                            echo "📊 Generating JaCoCo for: ${svc}"
                            dir(svc) {
                                jacoco(
                                    execPattern: 'target/jacoco.exec',
                                    classPattern: "target/classes",
                                    sourcePattern: "src/main/java",
                                    exclusionPattern: "**/test/**",
                                    minimumLineCoverage: '70',
                                    changeBuildStatus: true
                                )
                            }
                        }
                    }
                }
            }
        }

        stage('Build') {
            agent any
            when {
                expression { return env.SERVICES != null && env.SERVICES != "" }
            }
            steps {
                script {
                    def services = env.SERVICES.split(',')
                    services.each { svc ->
                        echo "🔨 Building: ${svc}"
                        dir(svc) {
                            sh '../mvnw clean package -DskipTests -T 1C'
                        }
                    }
                }
            }
        }
    }

    post {
    success {
        script {
            def commitId = env.GIT_COMMIT
            echo "✅ Sending 'success' to GitHub: ${commitId}"
            withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                sh """
                    curl -X POST \
                        -H "Authorization: token \$GITHUB_TOKEN" \
                        -H "Accept: application/vnd.github.v3+json" \
                        -d '{\"state\":\"success\",\"description\":\"Build passed\",\"context\":\"ci/jenkins-pipeline\",\"target_url\":\"${env.BUILD_URL}\"}' \
                        https://api.github.com/repos/phucvu0210/spring-petclinic-microservices/statuses/${commitId}
                """
            }
        }
    }
    failure {
        script {
            def commitId = env.GIT_COMMIT
            echo "❌ Sending 'failure' to GitHub: ${commitId}"
            withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                sh """
                    curl -X POST \
                        -H "Authorization: token \$GITHUB_TOKEN" \
                        -H "Accept: application/vnd.github.v3+json" \
                        -d '{\"state\":\"failure\",\"description\":\"Build failed\",\"context\":\"ci/jenkins-pipeline\",\"target_url\":\"${env.BUILD_URL}\"}' \
                        https://api.github.com/repos/phucvu0210/spring-petclinic-microservices/statuses/${commitId}
                """
            }
        }
    }
}
}