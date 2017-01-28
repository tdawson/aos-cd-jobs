// TODO Replace flow control (when available) with:
// https://jenkins.io/blog/2016/12/19/declarative-pipeline-beta/
def try_wrapper(failure_func, f) {
    try {
        f.call()
    } catch(err) {
        failure_func(err)
        // Re-throw the error in order to fail the job
        throw err
    }
}

def mail_success(version) {
    mail(
        to: 'jupierce@redhat.com',
        subject: "[aos-devel] New AtomicOpenShift Puddle for OSE: ${version}",
        body: """\
v${version}
Images have been built for this puddle
Images have been pushed to registry.ops
Puddles have been synched to mirrors


Jenkins job: ${env.BUILD_URL}
""");
}

def mail_failure = { err ->
    mail(
        to: 'jupierce@redhat.com',
        subject: "Error building OSE: ${OSE_MAJOR}.${OSE_MINOR}",
        body: """\
Encoutered an error while running merge-and-build.sh: ${err}


Jenkins job: ${env.BUILD_URL}
""");
}

def git_merge(branch, commit, commit_msg, merge_opts = '') {
    sh("""\
git config user.name jenkins
git config user.email jenkins@example.com
git checkout ${branch}
git merge ${merge_opts} '${commit}' -m '${commit_msg}'
""")
}

node {
    properties([[
        $class: 'ParametersDefinitionProperty',
        parameterDefinitions: [
            [
                $class: 'hudson.model.StringParameterDefinition',
                defaultValue: '',
                description: 'OSE Major Version',
                name: 'OSE_MAJOR',
            ],
            [
                $class: 'hudson.model.StringParameterDefinition',
                defaultValue: '',
                description: 'OSE Minor Version',
                name: 'OSE_MINOR',
            ],
            [
                $class: 'hudson.model.StringParameterDefinition',
                name: 'OSE_REPO',
                description: 'OSE repository url',
                defaultValue: 'https://github.com/openshift/ose.git',
            ],
            [
                $class: 'hudson.model.StringParameterDefinition',
                name: 'ORIGIN_REPO',
                description: 'Origin repository url',
                defaultValue: 'https://github.com/openshift/origin.git',
            ],
            [
                $class: 'hudson.model.StringParameterDefinition',
                name: 'WEB_CONSOLE_REPO',
                description: 'Origin web console repository url',
                defaultValue:
                    'https://github.com/openshift/origin-web-console.git',
            ],
        ],
    ]])
    stage('Merge and build') {
        try_wrapper(mail_failure) {
            env.GOPATH = env.WORKSPACE + '/go'
            dir(env.GOPATH) { deleteDir() }
            sh 'go get github.com/jteeuwen/go-bindata'
            dir(env.GOPATH + '/src/github.com/openshift/origin-web-console') {
                git url: WEB_CONSOLE_REPO
                def v = "enterprise-${OSE_MAJOR}.${OSE_MINOR}"
                git_merge('master', "origin/${v}", "Merge master into ${v}")
            }
            dir(env.GOPATH + '/src/github.com/openshift/ose') {
                checkout(
                    $class: 'GitSCM',
                    branches: [[name: 'refs/remotes/origin/master']],
                    extensions:
                        [[$class: 'LocalBranch', localBranch: 'master']],
                    userRemoteConfigs: [
                        [name: 'upstream', url: "${ORIGIN_REPO}"],
                        [name: 'origin', url: "${OSE_REPO}"]])
                git_merge(
                    'master', 'upstream/master',
                    'Merge remote-tracking branch upstream/master',
                    '--strategy-option=theirs')
                sh 'GIT_REF=master COMMIT=1 hack/vendor-console.sh'
                sh 'tito tag --accept-auto-changelog'
                def v = readFile(file: 'origin.spec') =~ /Version:\s+([.0-9]+)/
                mail_success(v[0][1])
            }
        }
    }
}
