class Constants {
    static final String MASTER_BRANCH = 'main'

    static final String QA_BUILD = 'Debug'
    static final String RELEASE_BULD = 'Release'

    static final String INTERNAL_TRACK = 'internal'
    static final String RELEASE_TRACK = 'release'
}

def getBuildType(){
    switch (env.BRANCH_NAME){
        case Constants.MASTER_BRANCH:
            return Constants.RELEASE_BULD
        default:
            return Constants.QA_BUILD    
    }
}

def getTrackType(){
    switch (env.BRANCH_NAME){
        case Constants.MASTER_BRANCH:
            return Constants.RELEASE_TRACK
        default:
            return Contatns.INTERNAL_TRACK     
    }
}

def isDeployCandidate(){
    return ("${env.BRANCH_NAME}" =~ /(master))
}

agent { dockerfile true }

environment {
    appName = 'CICDProject'

    KEY_PASSWORD= credentials('keyPasswordCICD')
    KEY_ALIAS = credentials('CICDProject')
    KEYSTORE = credentials('keystore')
    STORE_PASSWORD = credentials('keyPasswordCICD')
}

stage('Run Tests'){
    steps {
        echo 'Running Tests'
        script {
            VARIANT = getBuildType()
            sh "./gradlew test${VARIANT}UnitTest"
        }
    }
}

stage('Build Bundle'){
    when { expression {return isDeployCandidate() } }
    steps {
        echo 'Building'
        script {
            VARIANT = getBuildType()
            sh "./gradlew -PstorePass=${STORE_PASSWORD}" -Pkeystore=${KEYSTORE} -Palias=${KEY_ALIAS} -Pkeypass=${KEY_PASSWORD} bundle${VARIANT}
        }
    }
}

stage('Deploy App to Store'){
    when { expression { return isDeployCandidate }}
    steps {
        echo 'Deploying'
        script{
            VARIANT = getBuildType()
            TRACK = getTrackType()

            if(TRACK === Constants.RELEASE_TRACK){
                timeout(time: 5, unit: 'MINUTES'){
                    input "Procceed with deployment to ${TRACK}?"
                }

                try {
                    CHANGELOG = readFile(file: 'CHANGELOG.txt')
                } catch (err){
                    echo "Issue reading CHANGELOG.txt file ${err.localizedMessage}"
                    CHANGELOG = ""
                }

                androidApkUpload googleCredentialsId : 'play-store-credentials',
                    filesPattern: "**/outputs/bundle/${VARIANT.toLowerCase()}/*.aab",
                    trackName: Track,
                    recentChangeList: [[language: 'en-US, text: CHANGELOG']]
            }
        }
    }
}