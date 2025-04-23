pipeline {
    agent { label 'VMAPS_AGILE23' }

    environment {
        http_proxy = 'http://internet.com:83'
        https_proxy = 'http://internet.fod.com:83'
        no_proxy = '127.0.0.1,localhost,.fod.com,fod.com'
    }

    parameters {
        choice(name: 'InputMode', choices: ['playbookIds', 'filterDateRange'], description: 'Choose how to fetch test results')
        string(name: 'ScriptToRun', description: 'Playbook IDs — only used if InputMode is playbookIds')
        string(name: 'filterId', description: 'Filter ID — only used if InputMode is filterDateRange')
        string(name: 'start_date', defaultValue: '', description: 'Start Date (yyyy-MM-dd)')
        string(name: 'end_date', defaultValue: '', description: 'End Date (yyyy-MM-dd)')
    }

    stages {

        // Stage 1: Handle input and validate
        stage('Validating Input Parameters') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'ecu_test_token', variable: 'testguidetoken')]) {
                        TOKEN = "$testguidetoken"

                        if (params.InputMode == 'playbookIds') {
                            def inputValues = params.ScriptToRun
                            def playbookinputRaw = inputValues.split(',').join(' ')
                            def cleaned = playbookinputRaw.replaceAll(" ", "")

                            if (cleaned.isNumber()) {
                                playbookinput = playbookinputRaw
                                echo "Using Playbook ID input: ${playbookinput}"
                            } else {
                                error("Invalid Playbook ID input")
                            }
                        } else if (params.InputMode == 'filterDateRange') {
                            if (params.filterId && params.start_date && params.end_date) {
                                echo "Using Filter-based input:"
                                echo "Filter ID: ${params.filterId}, Start: ${params.start_date}, End: ${params.end_date}"
                            } else {
                                error("Missing one or more of: filterId, start_date, end_date")
                            }
                        } else {
                            error("Unknown InputMode: ${params.InputMode}")
                        }
                    }
                }
            }
        }

        // Stage 2: Run the Python script to fetch test output
        stage('Fetch Test Output via Python') {
            steps {
                script {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jira_jt', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                        bat "pyenv local 3.8.3"
                        env.PYTHONPATH = "${env.WORKSPACE}\\PythonScripts\\Python_Modules"

                        def command = ""
                        if (params.InputMode == 'playbookIds' && playbookinput) {
                            command = "python PythonScripts/Python.py --auth_key ${TOKEN} --playbook_ids ${playbookinput}"
                        } else if (params.InputMode == 'filterDateRange') {
                            command = "python PythonScripts/Python.py --auth_key ${TOKEN} --filter_id ${params.filterId} --start_date ${params.start_date} --end_date ${params.end_date}"
                        }

                        if (command) {
                            def ScriptPythonOutput = bat(label: 'Run Python Fetch', script: command, returnStdout: true).trim()
                            def pathToJsonTemplate = "${env.WORKSPACE}\\test_output.json"
                            ParameterizedPlaybook = readFile file: pathToJsonTemplate
                            echo "Loaded output from: ${pathToJsonTemplate}"
                        } else {
                            error("No valid command to run")
                        }
                    }
                }
            }
        }

        // Stage 3: Parse test results and prepare JIRA input
        stage('Prepare Data for Ticketing') {
            steps {
                script {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jira_', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                        if (ParameterizedPlaybook) {
                            def jsonSlurper = new groovy.json.JsonSlurper()
                            def jsonObject = jsonSlurper.parseText(ParameterizedPlaybook)
                            resultnewArray = []

                            jsonObject.tasks.each { taskItem ->
                                taskItem.test_cases.each { testCase ->
                                    if (testCase['result'] in ['FAIL']) {
                                        def tempTestName = "${testCase['test_name'].split("/")[-1].split("\\.pkg")[0]} FAIL ${testCase['failed_step']}"
                                        if (tempTestName.length() > 250) {
                                            tempTestName = tempTestName.substring(0, 249)
                                        }

                                        def task_id = taskItem['task_id']
                                        def url = testCase['url']
                                        def vcm_sw_version = testCase['vcm_sw_version'] ?: ""
                                        def global_id = testCase['global_id'] ?: ""

                                        def existing = resultnewArray.find { it.name == tempTestName }

                                        if (existing) {
                                            existing.count += 1
                                            if (!existing.task_ids.contains(task_id)) {
                                                existing.task_ids << task_id
                                                existing.urls << url
                                                existing.vcm_sw_version << vcm_sw_version
                                                existing.global_id << global_id
                                            }
                                        } else {
                                            resultnewArray << [
                                                name: tempTestName,
                                                count: 1,
                                                task_ids: [task_id],
                                                urls: [url],
                                                vcm_sw_version: [vcm_sw_version],
                                                global_id: [global_id]
                                            ]
                                        }
                                    }
                                }
                            }

                            echo "Formatted data ready for JIRA ticketing."
                        } else {
                            error("No test results to process")
                        }
                    }
                }
            }
        }

        // Stage 4: Create or update JIRA tickets
        stage('Create/Update JIRA Tickets') {
            steps {
                script {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jira_', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                        if (resultnewArray) {
                            echo "Creating JIRA tickets..."
                            IssueJiraCheckAll(USERNAME, PASSWORD, "CITISB", resultnewArray)
                        } else {
                            echo "No failed test cases to create tickets for."
                        }
                    }
                }
            }
        }
    }
}
