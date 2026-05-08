// =============================================================
// Jenkinsfile — YAS Monorepo CI Pipeline
// Adjusted to match the actual root pom.xml:
//   - Java 25 (not 21)
//   - JaCoCo report bound to verify phase → test stage runs mvn verify
//   - Full module list from <modules> in root pom.xml
//   - Sonar org/host inherited from pom.xml, only projectKey overridden
//   - Failsafe integration test reports published alongside surefire
// =============================================================

// All modules declared in the root pom.xml <modules> block.
// common-library is a shared lib with no runnable tests of its own —
// it is included so path changes there still trigger a pipeline run.
def JAVA_SERVICES = [
    'common-library', 'backoffice-bff', 'cart', 'customer',
    'delivery', 'inventory', 'location', 'media', 'order',
    'payment', 'payment-paypal', 'product', 'promotion',
    'rating', 'recommendation', 'sampledata', 'search',
    'storefront-bff', 'tax', 'webhook'
]

// Next.js frontend (not a Maven module — uses npm).
def NEXTJS_SERVICES = ['backoffice']

pipeline {
    agent any

    environment {
        // Your fork's SonarCloud organization slug.
        // Also update sonar.organization in the root pom.xml to match.
        SONAR_ORG = 'lmfao'
    }

    tools {
        // Install JDK 25 via the Eclipse Temurin installer plugin.
        // In Manage Jenkins → Tools → JDK, name it exactly 'JDK25'.
        jdk   'JDK25'
        maven 'Maven3.9'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 90, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // ── 1. Secret Scan ────────────────────────────────────
        // Runs before any build work. Fails immediately if secrets
        // or credentials are found committed in the source tree.
        stage('Secret Scan') {
            steps {
                sh '''
                    gitleaks detect \
                        --source . \
                        --report-format json \
                        --report-path gitleaks-report.json \
                        --redact \
                        --exit-code 1
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // ── 2. Detect Changed Services ────────────────────────
        // Three-dot diff (origin/main...HEAD) captures only the
        // commits unique to this branch — exactly what a PR changes.
        // Falls back to HEAD~1..HEAD for direct branch builds where
        // origin/main is identical to HEAD (e.g., a push to main itself).
        stage('Detect Changes') {
            steps {
                script {
                    sh 'git fetch origin main --depth=1 2>/dev/null || true'

                    def diff = ''
                    try {
                        diff = sh(
                            script: '''
                                if git rev-parse origin/main >/dev/null 2>&1 && \
                                   [ "$(git rev-parse HEAD)" != "$(git rev-parse origin/main)" ]; then
                                    git diff --name-only origin/main...HEAD
                                else
                                    git diff --name-only HEAD~1 HEAD 2>/dev/null || true
                                fi
                            ''',
                            returnStdout: true
                        ).trim()
                    } catch (e) {
                        echo "Change detection error — falling back to all services: ${e.message}"
                    }

                    def changedFiles = diff ? diff.split('\n').toList() : []

                    def filterServices = { List<String> services ->
                        if (!changedFiles) return services
                        def matched = services.findAll { svc ->
                            changedFiles.any { f -> f.startsWith("${svc}/") }
                        }
                        return matched ?: []
                    }

                    def javaHit = filterServices(JAVA_SERVICES)
                    def nextHit = filterServices(NEXTJS_SERVICES)

                    // Root-level changes (pom.xml, docker-compose, .github, etc.)
                    // affect the whole system — run all services.
                    if (javaHit.isEmpty() && nextHit.isEmpty() && !changedFiles.isEmpty()) {
                        echo 'Changes are outside service directories — running all services.'
                        javaHit = JAVA_SERVICES
                        nextHit = NEXTJS_SERVICES
                    }

                    env.CHANGED_JAVA = javaHit.join(',')
                    env.CHANGED_NEXT = nextHit.join(',')

                    echo "Java services   : ${env.CHANGED_JAVA ?: '(none)'}"
                    echo "Next.js services: ${env.CHANGED_NEXT ?: '(none)'}"
                }
            }
        }

        // ── 3. Test ───────────────────────────────────────────
        // Runs mvn verify (not mvn test) because the root pom.xml
        // binds the JaCoCo report goal to the verify phase. Running
        // only mvn test would never produce jacoco.xml, so the
        // Coverage plugin would have nothing to read.
        //
        // verify also runs integration tests via maven-failsafe-plugin
        // (files matching **/*IT.java). Their results land in
        // failsafe-reports and are published alongside surefire.
        //
        // Prerequisite: jacoco-maven-plugin must be moved from
        // <pluginManagement> into <plugins> in the root pom.xml.
        stage('Test') {
            steps {
                script {
                    def stages = [failFast: false]

                    env.CHANGED_JAVA.tokenize(',').each { svc ->
                        def service = svc.trim()
                        if (!service) return
                        stages["Test · ${service}"] = {
                            dir(service) {
                                sh 'mvn verify -B -Dmaven.test.failure.ignore=true'

                                // Unit test results (maven-surefire-plugin)
                                junit testResults: '**/target/surefire-reports/*.xml',
                                      allowEmptyResults: true

                                // Integration test results (maven-failsafe-plugin)
                                junit testResults: '**/target/failsafe-reports/*.xml',
                                      allowEmptyResults: true

                                // Coverage gate — build fails if line coverage < 70%.
                                // jacoco.xml is written to target/site/jacoco/ by the
                                // report goal during the verify phase above.
                                recordCoverage(
                                    tools: [[
                                        parser:  'JACOCO',
                                        pattern: '**/target/site/jacoco/jacoco.xml'
                                    ]],
                                    qualityGates: [[
                                        threshold:   70.0,
                                        metric:      'LINE',
                                        baseline:    'PROJECT',
                                        criticality: 'FAILURE'
                                    ]]
                                )
                            }
                        }
                    }

                    env.CHANGED_NEXT.tokenize(',').each { svc ->
                        def service = svc.trim()
                        if (!service) return
                        stages["Test · ${service}"] = {
                            dir(service) {
                                sh '''
                                    npm ci
                                    npm test -- \
                                        --coverage \
                                        --watchAll=false \
                                        --ci \
                                        --reporters=default \
                                        --reporters=jest-junit
                                '''
                                junit testResults: 'junit.xml',
                                      allowEmptyResults: true
                            }
                        }
                    }

                    if (stages.size() > 1) {
                        parallel stages
                    } else {
                        echo 'No services affected — skipping Test stage.'
                    }
                }
            }
        }

        // ── 4. Build ──────────────────────────────────────────
        // mvn verify already ran package as part of the lifecycle,
        // so this stage re-runs package with tests skipped purely to
        // produce a clean artifact and keep Test and Build as distinct
        // visible stages (as required). Extend this stage later for
        // Docker image builds or registry pushes.
        stage('Build') {
            steps {
                script {
                    def stages = [failFast: false]

                    env.CHANGED_JAVA.tokenize(',').each { svc ->
                        def service = svc.trim()
                        if (!service) return
                        stages["Build · ${service}"] = {
                            dir(service) {
                                sh 'mvn package -B -DskipTests -DskipITs'
                                archiveArtifacts artifacts: 'target/*.jar',
                                                 allowEmptyArchive: true
                            }
                        }
                    }

                    env.CHANGED_NEXT.tokenize(',').each { svc ->
                        def service = svc.trim()
                        if (!service) return
                        stages["Build · ${service}"] = {
                            dir(service) {
                                sh 'npm ci && npm run build'
                            }
                        }
                    }

                    if (stages.size() > 1) {
                        parallel stages
                    } else {
                        echo 'No services affected — skipping Build stage.'
                    }
                }
            }
        }

        // ── 5. Code Quality — SonarCloud ─────────────────────
        // sonar.organization and sonar.host.url are already declared
        // in the root pom.xml and inherited by every module — no need
        // to pass them on the command line. Only sonar.projectKey is
        // overridden here to give each service its own SonarCloud project.
        //
        // Before this works:
        //   1. Update sonar.organization in root pom.xml to your fork's org.
        //   2. Create a project per service in SonarCloud, or enable
        //      auto-provisioning in your org settings.
        //   3. The SonarCloud token lives in the Jenkins SonarQube server
        //      config (Manage Jenkins → System), not as a separate credential.
        stage('Code Quality') {
            steps {
                script {
                    def stages = [failFast: false]

                    env.CHANGED_JAVA.tokenize(',').each { svc ->
                        def service = svc.trim()
                        if (!service) return
                        stages["SonarCloud · ${service}"] = {
                            dir(service) {
                                withSonarQubeEnv('SonarCloud') {
                                    sh """
                                        mvn sonar:sonar -B \
                                            -Dsonar.projectKey=${env.SONAR_ORG}_yas-${service} \
                                            -Dsonar.java.source=25 \
                                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                                    """
                                }
                            }
                        }
                    }

                    if (stages.size() > 1) {
                        parallel stages
                    } else {
                        echo 'No Java services affected — skipping SonarCloud.'
                    }
                }
            }
        }

        // ── 6. Vulnerability Scan — Snyk ─────────────────────
        // Scans Maven dependencies for known CVEs per service.
        // Results are archived as JSON artifacts.
        // Non-blocking (|| true) by default — remove once baseline is clean.
        stage('Vulnerability Scan') {
            steps {
                script {
                    def stages = [failFast: false]

                    env.CHANGED_JAVA.tokenize(',').each { svc ->
                        def service = svc.trim()
                        if (!service) return
                        stages["Snyk · ${service}"] = {
                            dir(service) {
                                withCredentials([
                                    string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')
                                ]) {
                                    sh """
                                        snyk test \
                                            --all-projects \
                                            --severity-threshold=high \
                                            --json-file-output=snyk-report-${service}.json \
                                            || true
                                    """
                                    archiveArtifacts artifacts: "snyk-report-${service}.json",
                                                     allowEmptyArchive: true
                                }
                            }
                        }
                    }

                    if (stages.size() > 1) {
                        parallel stages
                    } else {
                        echo 'No services affected — skipping Snyk scan.'
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "✅ Pipeline passed — branch: ${env.BRANCH_NAME}"
        }
        unstable {
            echo "⚠️  Pipeline unstable (test failures or coverage below 70%) — branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline failed — branch: ${env.BRANCH_NAME}. Check stage logs above."
        }
    }
}
