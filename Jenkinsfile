pipeline {
    agent any
    
    triggers {
        pollSCM('H/5 * * * *') // Проверять изменения каждые 5 минут
    }
    
    environment {
        MYSQL_HOST = 'mysql-rfam-public.ebi.ac.uk'
        MYSQL_USER = 'rfamro'
        MYSQL_PORT = '4497'
        MYSQL_DATABASE = 'Rfam'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Detect SQL Changes') {
            steps {
                script {
                    // Получаем список измененных файлов
                    def changedFiles = getChangedFiles()
                    def sqlFiles = changedFiles.findAll { it.endsWith('.sql') }
                    
                    if (sqlFiles.isEmpty()) {
                        currentBuild.result = 'SUCCESS'
                        echo 'No SQL files changed. Skipping execution.'
                        return
                    }
                    
                    // Сохраняем список измененных SQL файлов для следующих стадий
                    env.CHANGED_SQL_FILES = sqlFiles.join(',')
                    echo "Changed SQL files: ${env.CHANGED_SQL_FILES}"
                }
            }
        }
        
        stage('Execute SQL Queries') {
            when {
                expression { return env.CHANGED_SQL_FILES != null }
            }
            steps {
                script {
                    def sqlFiles = env.CHANGED_SQL_FILES.split(',')
                    
                    sqlFiles.each { sqlFile ->
                        stage("Execute ${sqlFile}") {
                            echo "Executing SQL script: ${sqlFile}"
                            
                            // Выполняем SQL запрос и сохраняем результат
                            def result = executeSQLQuery(sqlFile)
                            
                            // Сохраняем результат как артефакт сборки
                            writeFile file: "result_${sqlFile}.txt", text: result
                            archiveArtifacts artifacts: "result_${sqlFile}.txt", fingerprint: true
                            
                            // Выводим результат в консоль
                            echo "Result of ${sqlFile}:\n${result}"
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs() // Очистка рабочей директории
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Check console output at ${env.BUILD_URL}",
                to: "developer@company.com"
            )
        }
    }
}

// Функция для получения списка измененных файлов
def getChangedFiles() {
    def changedFiles = []
    
    if (currentBuild.changeSets) {
        currentBuild.changeSets.each { changeSet ->
            changeSet.items.each { entry ->
                entry.affectedFiles.each { file ->
                    changedFiles.add(file.path)
                }
            }
        }
    }
    
    return changedFiles.unique()
}

// Функция для выполнения SQL запроса
def executeSQLQuery(sqlFile) {
    try {
        // Читаем содержимое SQL файла
        def sqlContent = readFile(file: sqlFile)
        
        // Выполняем запрос через mysql клиент
        def result = sh(
            script: """
                mysql \
                --user=${env.MYSQL_USER} \
                --host=${env.MYSQL_HOST} \
                --port=${env.MYSQL_PORT} \
                --database=${env.MYSQL_DATABASE} \
                --batch \
                --silent \
                --execute "${sqlContent.replace('"', '\\"')}"
            """,
            returnStdout: true
        ).trim()
        
        return result ?: "No results returned"
        
    } catch (Exception e) {
        return "ERROR executing SQL query: ${e.getMessage()}"
    }
}
