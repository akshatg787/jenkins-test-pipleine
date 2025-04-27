pipeline {
    agent { label 'VMAPS_AGILE23' }

    environment {
        http_proxy = 'http://internet.fod.com:83'
        https_proxy = 'http://internet.fod.com:83'
        no_proxy = '127.0.0.1,localhost,.fod.com,ford.com'
    }

    stages {
        stage('Validating input parameters') {
            steps {
                script {
                    import groovy.json.JsonSlurper
                    def playbookinput = ""
                    def ParameterizedPlaybook = ""
                    def resultnewArray = []

                    withCredentials([string(credentialsId: 'ecu_test_token', variable: 'testguidetoken')]) {
                        TOKEN = "$testguidetoken"
                        def inputValues = "${params.SelectChoise}"
                        def inputValuesselected = params.ScriptToRun.split(',')
                        def playbooksids = inputValuesselected.join(' ')
                        def valid_playbookids = playbooksids.replaceAll(" ", "")
                        
                        if (inputValues == 'playbookIds' && valid_playbookids.isNumber()) {
                            playbookinput = playbooksids
                            echo "Input is ${playbookinput}"
                        } else {
                            error("Bad input: Please check the input parameters.")
                        }
                    }
                }
            }
        }

        // next stages...
    }
}
