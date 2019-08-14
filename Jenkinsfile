#!groovy

import net.sf.json.JSONArray;
import net.sf.json.JSONObject;

pipeline {
    agent any
    environment {

        // For teste purpose only (need to be set always to TRUE! )
        RUN_PRE_BUILD = true
        RUN_POST_BUILD = true
        RUN_COMPILE = true
        RUN_CI = true
        S3_BUCKET_ARTIFACT = "cdt-devops-tools-lambda-functions-artifacts"
        S3_BUCKET_TEMPLATE = "cdt-devops-tools-lambda-functions-template"
        //path = "splt_init"
        //newVersion = "${newVersion}"

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
                            env["RUN_DEPLOY_DEV"]=true
                            current_commit_message = sh(script: '''
                                git rev-list --format=%B --max-count=1 HEAD |head -2 |tail -1
                            ''', returnStdout: true).trim()

                            if (current_commit_message == 'Prepare for next Release') {
                                currentBuild.result = 'ABORTED'
                                error('Parando build por ser um commit de CI.')
                            }
                        }
                    }
                }

                stage('check-commits-behind') {
                    steps {
                        checkCommitBehind()
                    }
                }

            }

        }

        stage('build'){
            parallel{
                stage('notify') {
                    steps {
                        echo sh(returnStdout: true, script: 'env')
                        //notifyBuild('STARTED')
                    }
                }

                stage('compile') {
                    steps {
                        script{
                            echo "build"
                            //sh 'npm install --save'
                            //sh 'zip -r p2pIssuerTransfer.zip .'
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

        stage(''){
            parallel{
                stage("dev"){
                    stages{
                        stage('deploy: dev') {
                            when {
                                    environment name: 'RUN_DEPLOY_DEV', value: 'true'
                                }       
                                steps {
                                        echo "Iniciando deploy no ambiente de DEV"
                                }
                        }
                            
                        stage('tests: dev'){
                            when {
                                environment name: 'RUN_DEPLOY_DEV', value: 'true'
                            }    
                            steps{
                                script{
                                    // echo "Testes DEV"
                                    // if(false){
                                    //     echo "Testes DEV"
                                    // }else{
                                    //     env["ROLLBACK"]=true
                                    //     currentBuild.result = 'FAILURE'
                                    // }
                                    try{
                                        echo "Testes DEV"
                                        sh('exit 1')
                                    }catch(Exception e){
                                        env["ROLLBACK"]=true
                                        currentBuild.result = 'FAILURE'
                                    }
                                }
                            }
                        }

                        stage('pull-request'){
                                when {
                                    environment name: 'RUN_DEPLOY_DEV', value: 'true'
                                    environment name: 'ROLLBACK', value: 'false'
                                }              
                                steps{
                                    echo "Creating pull request to ble"
                                    createPullRequest('master')
                                }
                        }
                    }                            
                }
                stage("hml"){
                    stages{
                        stage('deploy: hml'){
                            when {
                                environment name: 'RUN_DEPLOY_HML', value: 'true'
                            }            
                            steps{
                                script{
                                    echo "Deploy HML"
                                }
                            }
                        }

                        stage('tests: hml'){
                            when {
                                environment name: 'RUN_DEPLOY_HML', value: 'true'
                            }            
                            steps{
                                script{
                                    try{
                                        echo "Testes HML"
                                    }catch(Exception e){
                                        env['ROLLBACK'] = true
                                    }
                                }
                            }
                        }
                    }
                    
                }
                stage("prd"){
                    stages{
                        stage('deploy: prd'){
                            when {
                                environment name: 'RUN_DEPLOY_PRD', value: 'true'
                            }            
                            steps{
                                script{
                                    echo "Deploy PRD"
                                }
                            }
                        }

                            stage('merge-back') {
                            when {
                                environment name: 'RUN_DEPLOY_PRD', value: 'true'
                            }            
                            steps {
                                script{
                                    merge_back()
                                }
                            }
                        }

                        stage('delete-branch'){
                            when {
                                environment name: 'RUN_DEPLOY_PRD', value: 'true'
                            }            
                            steps{
                                script{
                                    echo "Delete Branch"
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('rollback'){
            // when{
            //     environment name: 'ROLLBACK', value: 'true'
            // }
            steps{
                script{
                    if (currentBuild.result != 'SUCCESS'){
                        echo "Efetuando rollback por falhas no teste"
                    }
                    
                }
            }
        }

    }
    post {
        success {
            //notifyBuild('SUCCESSFUL')
            echo "success"
        }
        failure {
            //notifyBuild('FAILED')
            echo "failure"
            // TO-DO delete cloud formation from S3
        }
        always {
            deleteDir() // Must delete after build, random errors occurs reusing workspace
        }
    }
}


def notifyBuild(String buildStatus = 'STARTED') {
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
        field.put('value', 'p2pIssuerTransfer');
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

def createPullRequest(String branchBase) {
    script{
        //env['current_commit_message'] = sh(script: ''' 
        //            git rev-list --format=%B --max-count=1 HEAD |head -2 |tail -1 
        //''', returnStdout: true)
        sh(script: '''
                git clone git@github.com:cdt-baas/github-backup.git
                getToken=`python3 github-backup/getGitToken.py`
                echo ${current_commit_message}
                current_commit_message=`git rev-list --format=%B --max-count=1 HEAD |head -2 |tail -1`
                curl -d '{ "title": "Pull Request automate pipeline", "body": "'"${current_commit_message}"'", "head": "develop", "base": "master" }' -H "Content-Type: application/json" -X POST https://api.github.com/repos/cdt-baas/p2ptransfer-serverbased/pulls?access_token=${getToken}
        ''', returnStdout: true)

        sh "git checkout ${BRANCH_NAME}"
        //createPullRequest = "hub pull-request -m '${current_commit_message}' -b ${branchBase} -h '${BRANCH_NAME}'"
        createPullRequest = "hub pull-request -m '${current_commit_message}' -b master -h '${BRANCH_NAME}'"
        echo "Running: ${createPullRequest}"
        pullRequestUrl = sh(returnStdout: true, script: createPullRequest).trim()
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

def updateVersion(boolean isMaster){
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

def validStats() {
    script {
        def stats = sh(script: '''
                        aws cloudformation describe-stacks --stack-name p2pIssuerTransfer --profile ${profile} | jq '.Stacks[0].StackStatus' | sed -e 's/\"//g'
                             ''', returnStdout: true).trim()
        if ( stats == 'UPDATE_COMPLETE') {
             echo "Success"
        }
        else if (stats == 'CREATE_COMPLETE') {
             echo "Success"
        }
        else if (stats == 'OK') {
            echo "Success"
        }
        else {
            currentBuild.result = 'FAILURE'       
            error("Problema ao realizar o deploy")
        }
    }
}
