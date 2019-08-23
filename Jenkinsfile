#!groovy

import net.sf.json.JSONArray;
import net.sf.json.JSONObject;

/*
New Jenkinfile with CI
Template: Node.js
*/

pipeline {
    agent any
    environment {

        // For teste purpose only (need to be set always to TRUE! )
        RUN_PRE_BUILD = true
        RUN_POST_BUILD = true
        RUN_COMPILE = true
        RUN_CI = true /*If false, disable CI*/
        S3_BUCKET_ARTIFACT = "cdt-devops-tools-lambda-functions-artifacts"
        S3_BUCKET_TEMPLATE = "cdt-devops-tools-lambda-functions-template"
        PATH_DEPLOY = "p2pIssuerTransfer"
        ARCHITETURE = "Serverless"
        JOB_ID = "06ba7d2a-6ee8-4c6b-ae9f-4c1b7714c850" //JobId do rundeck

    }
    options {
        //skipDefaultCheckout()
        // Only keep the 50 most recent builds
        buildDiscarder(logRotator(numToKeepStr: '50'))
        timeout(time: 20, unit: 'MINUTES')
        // disableConcurrentBuilds()
    }



    stages {
        stage('pre-build'){
            when {
                environment name: 'RUN_PRE_BUILD', value: 'true'
            }
            parallel{
                stage('check-commit-message') {
                    steps {
                        script {
                            current_commit_message = "hein"
                        }
                    }
                }

                stage('check-commits-behind') {
                    steps {
                        echo "check"
                    }
                }

                stage('set-envs') {
                    steps {
                        script {


                            if (BRANCH_NAME.equals("master")) {
                                echo "***** PERFORMING STEPS ON MASTER *****"
                                env['environment'] = "prd"
                                env['RUN_DEPLOY_PRD'] = true
                                //update_version(true)
                            } else if (BRANCH_NAME.startsWith("PR")) {
                                echo "***** PERFORMING STEPS ON PR *****"
                                env['environment'] = "hml"
                                env['RUN_DEPLOY_HML'] = true
                                //version_code_tag()
                                //env['newVersion'] = env['bumpci_tag']
                            } else if (BRANCH_NAME.startsWith("hotfix")) {
                                echo "***** PERFORMING STEPS ON PR *****"
                                env['RUN_HOTFIX'] = true
                                //update_version(false)
                            }
                            else {
                                echo "***** PERFORMING STEPS ON ANY BRANCH *****"
                                env['environment'] = "dev"
                                env['RUN_DEPLOY_DEV'] = true
                                //update_version(false)
                            }
                        }

                        echo "***** FINISHED PRE-BUILD STEP *****"
                    }
                } 
            }

        }

        stage('test-infra'){
            parallel{
                stage('cfn-lint'){
                    steps{
                        script{
                            catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                                echo "Not implemented"
                            }
                        }
                    }
                    
                }
                stage('cfn-security'){
                    steps{
                        script{
                            def status = 0
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE'){ 
                                //status = sh(script: "/usr/local/bin/cfn_nag_scan --input-path cloudformation/template/cloudformation.yml",  returnStatus:true)
                                status = sh(script: "command not exist",  returnStatus:true)
                                if(status != 0){
                                    //notify_build('INFRA-FAILED')
                                    sh "exit ${status}"
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('build'){
            parallel{
                stage('notify') {
                    steps {
                        echo sh(returnStdout: true, script: 'env')
                        //notify_build('STARTED')
                    }
                }

                stage('compile') {
                    steps {
                        script{
                            //sh 'npm install --save --quiet --silent --loglevel error'
                            //sh 'npm install --quiet --no-progress --silent'
                            //sh 'zip -r p2pIssuerTransfer.zip .'
                            echo "compile"
                        }
                    }
                } 
            }
        }

        stage('package-sam') {
            steps {
                script{
                    //sh 'aws cloudformation package --template-file cloudformation/template/cloudformation.yml --s3-bucket ${S3_BUCKET_ARTIFACT}  --output-template-file cloudformation/output/cloudformation.yml'
                    //env['fileOutput'] = 'cloudformation.yml'
                    echo "package"
                }
            }
        }

        stage('upload-iaac-to-s3') {
            steps {
                script{
                    echo "upload"
                }
            }
        }

        stage('deployment'){
            parallel{
                stage("dev"){
                    stages{
                         stage('deploy: dev') {
                            when {
                                environment name: 'RUN_DEPLOY_DEV', value: 'true'
                            }       
                            steps {
                                echo "deploy"
                                //deploy('dev')
                            }
                        }/*end stage deploy: dev*/
                        stage('tests: dev'){
                            when {
                                environment name: 'RUN_DEPLOY_DEV', value: 'true'
                            }    
                            steps{
                                script{
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                                        tests('hml')
                                        sh "exit 0" //test purpose - remove whent tests is really running
                                    }
                                    if(currentBuild.result == 'UNSTABLE'){
                                        echo "Tests failed! I will rollback"
                                        env["ROLLBACK"]=true
                                    }
                                }
                            }
                        }/*end stage tests: dev*/

                         stage('pull-request'){
                            when {
                                environment name: 'RUN_DEPLOY_DEV', value: 'true'
                                environment name: 'RUN_CI', value: 'true'
                                not{
                                    environment name: 'ROLLBACK', value: 'true'       
                                }
                                not{
                                    environment name:  'RUN_HOTFIX', value: 'true'
                                }
                            }              
                            steps{
                                echo "Creating pull request to master"
                                create_pull_request('master')
                            }
                        } /*end stage pull-request*/
                    }/*end stages*/
                }/*end stage dev*/

                stage("hml"){
                    when{
                        environment name: 'RUN_CI', value: 'true'
                    }
                    stages{
                        stage('deploy: hml'){
                            when {
                                environment name: 'RUN_DEPLOY_HML', value: 'true'
                            }            
                            steps{
                                //deploy('hml')
                                echo "deploy"
                            }
                        }/*end stage deploy-hml*/

                        stage('tests: hml'){
                            when {
                                environment name: 'RUN_DEPLOY_HML', value: 'true'
                            }            
                            steps{
                                script{
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                                        tests('hml')
                                        sh "exit 0" //test purpose - remove whent tests is really running
                                    }
                                    if(currentBuild.result == 'UNSTABLE'){
                                        echo "Tests failed! I will rollback"
                                        env["ROLLBACK"]=true
                                    }
                                }
                            }
                        }/*end stage test-hml*/
                    }/*end stages*/
                }/*end stage html*/
                stage("prd"){
                    when{
                        environment name: 'RUN_CI', value: 'true'
                    }
                    stages{
                        stage('deploy: prd'){
                            when {
                                environment name: 'RUN_DEPLOY_PRD', value: 'true'
                            }            
                            steps{
                                //deploy('prd')
                                echo "deploy"
                            }
                        }/*end stage deploy-prd*/

                        stage('merge-back') {
                            when {
                                environment name: 'RUN_DEPLOY_PRD', value: 'true'
                            }            
                            steps {
                                merge_back()
                            }
                        }/*end stage merge-back*/

                        stage('delete-branch'){
                            when {
                                environment name: 'RUN_DEPLOY_PRD', value: 'true'
                            }            
                            steps{
                                delete_branch()
                            }
                        }/*end stage delete-branch*/
                    }/*end stages*/
                }/*end stage prd*/
                stage('hotfix'){
                    stages{
                        stage('pull-request'){
                            when{
                                environment name:  'RUN_HOTFIX', value: 'true'
                            }
                            steps{
                                create_pull_request('master') 
                            }
                        }
                    }/*end stage pull-request*/
                }/*end stage hotfix*/
            }/*end stage parallel*/
        }/*end stage deployment*/

        stage('rollback'){
            when{
                 environment name: 'ROLLBACK', value: 'true'
            }
            steps{
                rollback()
            }
        }/*end stage rollback*/
    }/*end stage rollback*/
    post {
        success {
            //notify_build('SUCCESSFUL')
            echo "success"
        }
        failure {
            //notify_build('FAILED')
            echo "failure"
            // TO-DO delete cloud formation from S3
        }
        always {
            //send_logstash()
            deleteDir() // Must delete after build, random errors occurs reusing workspace
        }
    }
}




def notify_build(String buildStatus = 'STARTED') {
    // build status of null means successful
    buildStatus = buildStatus ?: 'SUCCESSFUL'

    // Default values
    String colorCode = '#FF0000'
    String subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
    String summary = "${subject} \n (${env.BUILD_URL})  "
    String details = """<p>${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""


    JSONArray attachments = new JSONArray();
    JSONObject attachment = new JSONObject();

    // Override default values based on build status
    if (buildStatus == 'STARTED') {
        colorCode = '#FFFF00'
        attachment.put('text','Começando o build. Será que vai dar certo?')
        attachment.put('thumb_url','https://pm1.narvii.com/6358/a711cc85a012317c9a8d4576dc1ae6eea95b5d5b_hq.jpg')
    } else if (buildStatus == 'SUCCESSFUL') {
        colorCode = '#00FF00'
        attachment.put('text','Build com sucesso. Parabéns!')
        attachment.put('thumb_url','http://4.bp.blogspot.com/-1K6KC1_3mok/Vn2ZpDHbr-I/AAAAAAAALB0/Jfs12qjfd6M/s640/%25C3%2593reas%2B-%2BOpenBrasil.org.png')

        JSONArray fields = new JSONArray();
        JSONObject field = new JSONObject();

        field.put('title', 'URL Template S3');
        field.put('value', env['fileOutput']);
        fields.add(field);

        field = new JSONObject();

        field.put('title', 'Version');
        field.put('value', env['newVersion']);
        fields.add(field);

        field.put('title', 'Path');
        field.put('value', ${PATH_DEPLOY});
        fields.add(field);

        attachment.put('fields',fields);

	    //def list_type = ['DEV', 'UAT']

        //JSONArray actions = new JSONArray();

      	//JSONObject action = new JSONObject();
      	//action.put('name','datahub-deploy');
        //action.put('text','Ambientes');
        //action.put('type','select');
        //JSONArray options = new JSONArray();
        //for (i = 0; i < list_type.size(); i++) {
        //    JSONObject option = new JSONObject();
        //    String envCode = list_type[i];
        //    option.put('text', envCode);
        //    option.put('value', envCode);
        //    options.add(option);
        //}
      	//action.put('options',options);
      	//actions.add(action);

        //attachment.put('actions',actions);

    } else {
        attachment.put('text','Errrrouuuu. Please again!')
        attachment.put('thumb_url','https://3.bp.blogspot.com/-S4p87C9SzYo/V8-ODgnEdTI/AAAAAAAABrQ/iSk6oPIS8F4NwtuF2FlBPo99yztBqfFOACLcB/s1600/Atos%2Bmontanha.jpg')
        colorCode = '#FF0000'
    }


    String buildUrl = "${env.BUILD_URL}";
    attachment.put('title', subject);
    attachment.put('callback_id', buildUrl);
    attachment.put('title_link', buildUrl);
    attachment.put('fallback', subject);
    attachment.put('color', colorCode);

    attachments.add(attachment);

    // Send notifications
    echo attachments.toString();
    slackSend(attachments: attachments.toString())

    //emailext(
    //        subject: subject,
    //        body: details,
    //        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
    //)
}

def create_pull_request(String branchBase) {
    script{
        echo "Creating Pull Request to ${branchBase}"
        // sh(script: '''
        //         git clone git@github.com:cdt-baas/github-backup.git
        //         getToken=`python3 github-backup/getGitToken.py`
        //         echo ${current_commit_message}
        //         current_commit_message=`git rev-list --format=%B --max-count=1 HEAD |head -2 |tail -1`
        //         curl -d '{ "title": "Pull Request automate pipeline", "body": "'"${current_commit_message}"'", "head": "develop", "base": "master" }' -H "Content-Type: application/json" -X POST https://api.github.com/repos/cdt-baas/p2ptransfer-serverbased/pulls?access_token=${getToken}
        // ''', returnStdout: true)

        // sh "git checkout ${BRANCH_NAME}"
        // createPullRequest = "hub pull-request -m '${current_commit_message}' -b ${branchBase} -h '${BRANCH_NAME}'"
        // echo "Running: ${createPullRequest}"
        // pullRequestUrl = sh(returnStdout: true, script: createPullRequest).trim()
   }
}

def checkCommitBehind() {
    sh 'echo "Verifica se branch necessita de merge com master."'
    script {
        sh(script: '''set +x; set +e;
                      git config --add remote.origin.fetch +refs/heads/master:refs/remotes/origin/master
                      git fetch --no-tags
                      commitsBehind=$(git rev-list --left-right --count origin/master... |awk '{print $1}')
                      if [ "${commitsBehind}" -ne 0 ]
                      then
                        echo "Esta branch está ${commitsBehind} commits atrás da master!"
                        exit 1
                      else
                        echo "Esta branch não tem commits atrás da master."
                      fi''')
    }
}

def update_version(boolean isMaster){
    version_code_tag()
    def oldVersion = "${env.bumpci_tag}".tokenize('.')
    major = oldVersion[0].toInteger()
    minor = oldVersion[1].toInteger()
    patch = oldVersion[2].toInteger()

    if(isMaster){
      minor += 1
      patch = 0
    }else {
      patch += 1
    }
    env['newVersion'] = major + '.' + minor + '.' + patch

    bump_version_tag()

}

def version_code_tag() {
    echo "getting Git version Tag"
    script {
        sh "git fetch --tags"
        env['bumpci_tag'] = sh(script: '''
            current_tag=`git tag -n9 -l |grep version |awk '{print $1}' |sort -V |tail -1`
            if [[ $current_tag == '' ]]
              then
                current_tag=0.0.1
            fi
            echo ${current_tag}
            ''', returnStdout: true).trim()
    }
  }

  def bump_version_tag() {
    echo "Bumping version CI Tag"
    script {
        sh "git tag -a ${newVersion} -m version && git push origin refs/tags/${newVersion}"
    }
  }


def merge_back() {
    echo "doing merge-back"
//  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'rundeck-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
//    script {
//      sh 'curl -s -c rundeck.cookie "http://devops-rundeck.smiles.local.br/j_security_check?j_username=${USERNAME}&j_password=${PASSWORD}"'
//      env['job_id'] = sh(script: '''
//        curl -s -b rundeck.cookie "http://devops-rundeck.smiles.local.br/api/19/project/git/jobs?jobExactFilter=mergeback" |grep '<job ' |sed -r 's,.*id="([^"]*).*,\\1,' |tr -d '\n'
//      ''', returnStdout: true)
//      env['repo'] = sh(script: '''
//        git remote show -n origin | grep Fetch | sed -r 's,.*:(.*).git,\\1,' |tr -d '\n'
//      ''', returnStdout: true)
//      sh 'curl -s -X POST -b rundeck.cookie "http://devops-rundeck.smiles.local.br/api/19/job/${job_id}/run?option.repo=${repo}"'
//    }
//  }
}

def deploy(environment){
    
    script {
        echo "Iniciando deploy no ambiente de ${environment}"
        env['profile'] = environment
        print_versions()
        env['current_commit_message'] = current_commit_message
        step([$class: "RundeckNotifier",
            includeRundeckLogs: true,
            jobId: "${JOB_ID}",
            nodeFilters: "",
            options: """
                        Arquitetura=${ARCHITETURE}
                        template=${fileOutput}
                        version=${newVersion}
                        path=${PATH_DEPLOY}
                    """,
            rundeckInstance: "rundeck.devtools.caradhras.io",
            shouldFailTheBuild: true,
            shouldWaitForRundeckJob: true,
            tags: "",
            tailLog: true])
    }
    valid_stats()
}

def tests(environment){
    script{
        echo "Running tests on: ${environment}"
    }
}

def delete_branch(){
    script(){
        echo "Deleting Branch"
    }
}

def valid_stats() {
    script {
        def stats = sh(script: '''
                        aws cloudformation describe-stacks --stack-name p2pIssuerTransfer --profile ${profile} | jq '.Stacks[0].StackStatus' | sed -e 's/\"//g'
                             ''', returnStdout: true).trim()
        if ( stats == 'UPDATE_COMPLETE' || stats == 'CREATE_COMPLETE' || stats == 'OK') {
             echo "Success"
        }
        else {
            currentBuild.result = 'FAILURE'       
            error("Problema ao realizar o deploy")
        }
    }
}

def print_versions(){
    script{
        def oldVersion = sh(script: '''
                            aws cloudformation describe-stacks --stack-name p2pIssuerTransfer --profile ${profile} --output text | grep Versao | cut -f3
                        ''',returnStdout: true).trim()

        //To ELK and Rollback
        echo "OLD_VERSION=${oldVersion}"
        echo "NEW_VERSION=${newVersion}"
    }
}

def rollback(){
    script{
        echo "Efetuando rollback por falhas no teste"
        currentBuild.result = 'UNSTABLE'
                    
    }
}

def checkPullRequest() {
    //echo 'Manual Promotion'
    // we need a first milestone step so that all jobs entering this stage are tracked an can be aborted if needed
    milestone 1
    // time out manual approval after ten minutes
    timeout(time: 10, unit: 'MINUTES') {
        input message: "Are you sure?"
    }
    // this will kill any job which is still in the input step
    milestone 2
}

def send_logstash(){
    logstashSend failBuild: true, maxLines: 1000
}