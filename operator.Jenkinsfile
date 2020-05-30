#!/usr/bin/env groovy

// PARAMETERS for this pipeline:
// def FORCE_BUILD = "false"
// def SOURCE_BRANCH = "master" // branch of source repo from which to find and sync commits to pkgs.devel repo

def SOURCE_REPO = "eclipse/che-operator" //source repo from which to find and sync commits to pkgs.devel repo
def GIT_PATH = "containers/codeready-workspaces-operator" // dist-git repo to use as target

def GIT_BRANCH = "crw-2.2-rhel-8" // target branch in dist-git repo, eg., crw-2.2-rhel-8
def SCRATCH = "false"
def PUSH_TO_QUAY = "true"
def QUAY_PROJECT = "operator" // also used for the Brew dockerfile params

def OLD_SHA=""

def buildNode = "rhel7-releng" // slave label
timeout(120) {
	node("${buildNode}"){ stage "Sync repos"
    wrap([$class: 'TimestamperBuildWrapper']) {
		  cleanWs()
	    withCredentials([string(credentialsId:'devstudio-release.token', variable: 'GITHUB_TOKEN'), 
        file(credentialsId: 'crw-build.keytab', variable: 'CRW_KEYTAB')]) {
		      checkout([$class: 'GitSCM',
		        branches: [[name: "${SOURCE_BRANCH}"]],
		        doGenerateSubmoduleConfigurations: false,
		        credentialsId: 'devstudio-release',
		        poll: true,
        extensions: [
          [$class: 'RelativeTargetDirectory', relativeTargetDir: "sources"],
        ],
		        submoduleCfg: [],
		        userRemoteConfigs: [[url: "https://github.com/${SOURCE_REPO}.git"]]])

  		      def BOOTSTRAP = '''#!/bin/bash -xe

# bootstrapping: if keytab is lost, upload to
# https://codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com/credentials/store/system/domain/_/
# then set Use secret text above and set Bindings > Variable (path to the file) as ''' + CRW_KEYTAB + '''
chmod 700 ''' + CRW_KEYTAB + ''' && chown ''' + USER + ''' ''' + CRW_KEYTAB + '''
# create .k5login file
echo "crw-build/codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com@REDHAT.COM" > ~/.k5login
chmod 644 ~/.k5login && chown ''' + USER + ''' ~/.k5login
 echo "pkgs.devel.redhat.com,10.19.208.80 ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAplqWKs26qsoaTxvWn3DFcdbiBxqRLhFngGiMYhbudnAj4li9/VwAJqLm1M6YfjOoJrj9dlmuXhNzkSzvyoQODaRgsjCG5FaRjuN8CSM/y+glgCYsWX1HFZSnAasLDuW0ifNLPR2RBkmWx61QKq+TxFDjASBbBywtupJcCsA5ktkjLILS+1eWndPJeSUJiOtzhoN8KIigkYveHSetnxauxv1abqwQTk5PmxRgRt20kZEFSRqZOJUlcl85sZYzNC/G7mneptJtHlcNrPgImuOdus5CW+7W49Z/1xqqWI/iRjwipgEMGusPMlSzdxDX4JzIx6R53pDpAwSAQVGDz4F9eQ==
" >> ~/.ssh/known_hosts

ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts

# see https://mojo.redhat.com/docs/DOC-1071739
if [[ -f ~/.ssh/config ]]; then mv -f ~/.ssh/config{,.BAK}; fi
echo "
GSSAPIAuthentication yes
GSSAPIDelegateCredentials yes

Host pkgs.devel.redhat.com
User crw-build/codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com@REDHAT.COM
" > ~/.ssh/config
chmod 600 ~/.ssh/config

# initialize kerberos
export KRB5CCNAME=/var/tmp/crw-build_ccache
kinit "crw-build/codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com@REDHAT.COM" -kt ''' + CRW_KEYTAB + '''
klist # verify working

hasChanged=0

SOURCEDOCKERFILE=${WORKSPACE}/sources/Dockerfile

# REQUIRE: skopeo
curl -L -s -S https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/master/product/updateBaseImages.sh -o /tmp/updateBaseImages.sh
chmod +x /tmp/updateBaseImages.sh
# don't bother to check/update if we're using a tag instead of a branch
if [[ "''' + SOURCE_BRANCH + '''" != "refs/tags/"* ]]; then
  cd ${WORKSPACE}/sources
    git checkout --track origin/''' + SOURCE_BRANCH + ''' || true
    export GITHUB_TOKEN=''' + GITHUB_TOKEN + ''' # echo "''' + GITHUB_TOKEN + '''"
    git config user.email "nickboldt+devstudio-release@gmail.com"
    git config user.name "Red Hat Devstudio Release Bot"
    git config --global push.default matching
    SOURCE_SHA=$(git rev-parse HEAD) # echo ${SOURCE_SHA:0:8}

    # can't yet generate a PR against eclipse/che-* repos, and can't push directly
    # so check for a new base image but DO NOT commit/push/pull-request
    /tmp/updateBaseImages.sh -b ''' + SOURCE_BRANCH + ''' -f ${SOURCEDOCKERFILE##*/} --nocommit
  cd ..
else
  cd ${WORKSPACE}/sources
    SOURCE_SHA=$(git rev-parse HEAD) # echo ${SOURCE_SHA:0:8}
  cd ..
fi

# fetch sources to be updated
GIT_PATH="''' + GIT_PATH + '''"
if [[ ! -d ${WORKSPACE}/target ]]; then git clone ssh://crw-build@pkgs.devel.redhat.com/${GIT_PATH} target; fi
cd ${WORKSPACE}/target
git checkout --track origin/''' + GIT_BRANCH + ''' || true
git config user.email crw-build@REDHAT.COM
git config user.name "CRW Build"
git config --global push.default matching
cd ..

'''
		      sh BOOTSTRAP

		      OLD_SHA = sh(script: '''#!/bin/bash -xe
		      cd ${WORKSPACE}/target; git rev-parse HEAD
		      ''', returnStdout: true)
		      println "Got OLD_SHA in target folder: " + OLD_SHA

          def SYNC_FILES = ".gitignore cmd deploy deploy.sh e2e Gopkg.lock Gopkg.toml olm pkg README.md templates vendor version"
		      sh BOOTSTRAP + '''

# rsync files in github to dist-git
for d in ''' + SYNC_FILES + '''; do
  if [[ -f ${WORKSPACE}/sources/${d} ]]; then
    rsync -zrlt ${WORKSPACE}/sources/${d} ${WORKSPACE}/target/${d}
  elif [[ -d ${WORKSPACE}/sources/${d} ]]; then
    # copy over the files
    rsync -zrlt ${WORKSPACE}/sources/${d}/* ${WORKSPACE}/target/${d}/
    # sync the directory and delete from target if deleted from source
    rsync -zrlt --delete ${WORKSPACE}/sources/${d}/ ${WORKSPACE}/target/${d}/
  fi
done
'''
          // OLD way
	  // def CRW_OPERATOR_IMAGE = "registry.redhat.io/codeready-workspaces/crw-2-rhel8-operator:latest"
		      // // relative to $WORKSPACE
		      // def files = findFiles(glob: 'target/deploy/**/*')
		      // files.each {
          //       println it.path
          //       // global string replacements in deploy scripts
          //       writeFile file: it.path, text: readFile(it.path)
          //           .replaceAll('quay.io/eclipse/che-operator:nightly', CRW_OPERATOR_IMAGE)
          //           .replaceAll('che/operator', 'codeready/operator')
          //           .replaceAll('che-operator', 'codeready-operator')
          //           .replaceAll('name: eclipse-che', 'name: codeready-workspaces')
          //           .replaceAll("cheImageTag: 'nightly'", "cheImageTag: ''")
          //           .replaceAll('/bin/codeready-operator', '/bin/che-operator')
		      // }

          //     // replacements in deploy/crds/org_v1_che_cr.yaml
          //     def cryaml = "target/deploy/crds/org_v1_che_cr.yaml"
          //     println cryaml
          //     writeFile file: cryaml, text: readFile(cryaml)
          //       .replaceAll("cheFlavor: ''", "cheFlavor: 'codeready'")
          //       .replaceAll("tlsSupport: .+", "tlsSupport: true")
          //       .replaceAll("devfileRegistryImage: 'quay.io/eclipse/.+'", "devfileRegistryImage: ''")
          //       .replaceAll("pluginRegistryImage: 'quay.io/eclipse/.+'", "pluginRegistryImage: ''")
          //       .replaceAll("identityProviderImage: 'quay.io/eclipse/.+'", "identityProviderImage: ''")
          //       .replaceAll("identityProviderAdminUserName: ''", "identityProviderAdminUserName: 'admin'")

          // NEW way - requires yq
		      sh BOOTSTRAP + '''
          sudo pip3 install yq
          jq --version
          yq --version
          ${WORKSPACE}/target/build/scripts/sync-che-operator-to-crw-operator.sh ${WORKSPACE}/sources/ ${WORKSPACE}/target/
          '''

          // get latest tags for the operator deployed images
          def opyaml = "target/deploy/operator.yaml"
          def images = [
            "registry.redhat.io/codeready-workspaces/server-rhel8",
            "registry.redhat.io/codeready-workspaces/pluginregistry-rhel8",
            "registry.redhat.io/codeready-workspaces/devfileregistry-rhel8",
            "registry.access.redhat.com/ubi8-minimal",
            "registry.redhat.io/rhel8/postgresql-96",
            "registry.redhat.io/rh-sso-7/sso74-openshift-rhel8",
            "registry.redhat.io/codeready-workspaces/pluginbroker-metadata-rhel8",
            "registry.redhat.io/codeready-workspaces/pluginbroker-artifacts-rhel8",
            "registry.redhat.io/codeready-workspaces/jwtproxy-rhel8"]
          result = readFile(opyaml)
          images.each() {
            latestTag = sh(returnStdout:true,script:"skopeo inspect docker://$it | jq -r .RepoTags[] | sort -V | grep -v 'source|latest' | tail -1").trim()
            echo "[INFO] Got image+tag: $it : $latestTag"
            result.replaceAll("$it:.+", "$it:" + latestTag)
          }
          writeFile file: opyaml, text: result

		      sh BOOTSTRAP + '''

cp -f ${SOURCEDOCKERFILE} ${WORKSPACE}/target/Dockerfile

# TODO should this be a branch instead of just master?
CRW_VERSION=`wget -qO- https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/master/dependencies/VERSION`
#apply patches
sed -i ${WORKSPACE}/target/Dockerfile \
  -e "s#FROM registry.redhat.io/#FROM #g" \
  -e "s#FROM registry.access.redhat.com/#FROM #g" \
  -e "s/# *RUN yum /RUN yum /g" \

METADATA='ENV SUMMARY="Red Hat CodeReady Workspaces ''' + QUAY_PROJECT + ''' container" \\\r
    DESCRIPTION="Red Hat CodeReady Workspaces ''' + QUAY_PROJECT + ''' container" \\\r
    PRODNAME="codeready-workspaces" \\\r
    COMPNAME="''' + QUAY_PROJECT + '''" \r
LABEL summary="$SUMMARY" \\\r
      description="$DESCRIPTION" \\\r
      io.k8s.description="$DESCRIPTION" \\\r
      io.k8s.display-name=\"$DESCRIPTION" \\\r
      io.openshift.tags="$PRODNAME,$COMPNAME" \\\r
      com.redhat.component="$PRODNAME-rhel8-$COMPNAME-container" \\\r
      name="$PRODNAME/$COMPNAME" \\\r
      version="'$CRW_VERSION'" \\\r
      license="EPLv2" \\\r
      maintainer="Nick Boldt <nboldt@redhat.com>" \\\r
      io.openshift.expose-services="" \\\r
      com.redhat.delivery.appregistry="false" \\\r
      usage="" \r'

echo -e "$METADATA" >> ${WORKSPACE}/target/Dockerfile

# push changes in github to dist-git
cd ${WORKSPACE}/target
if [[ \$(git diff --name-only) ]]; then # file changed
	OLD_SHA=\$(git rev-parse HEAD) # echo ${OLD_SHA:0:8}
	git add Dockerfile ''' + SYNC_FILES + '''
  /tmp/updateBaseImages.sh -b ''' + GIT_BRANCH + ''' --nocommit
  # note this might fail if we're syncing from a tag vs. a branch
  git commit -s -m "[sync] Update from ''' + SOURCE_REPO + ''' @ ${SOURCE_SHA:0:8}" Dockerfile ''' + SYNC_FILES + ''' || true
  git push origin ''' + GIT_BRANCH + ''' || true
  NEW_SHA=\$(git rev-parse HEAD) # echo ${NEW_SHA:0:8}
  if [[ "${OLD_SHA}" != "${NEW_SHA}" ]]; then hasChanged=1; fi
  echo "[sync] Updated pkgs.devel @ ${NEW_SHA:0:8} from ''' + SOURCE_REPO + ''' @ ${SOURCE_SHA:0:8}"
else
    # file not changed, but check if base image needs an update
    # (this avoids having 2 commits for every change)
    cd ${WORKSPACE}/target
    OLD_SHA=\$(git rev-parse HEAD) # echo ${OLD_SHA:0:8}
    /tmp/updateBaseImages.sh -b ''' + GIT_BRANCH + '''
    NEW_SHA=\$(git rev-parse HEAD) # echo ${NEW_SHA:0:8}
    if [[ "${OLD_SHA}" != "${NEW_SHA}" ]]; then hasChanged=1; fi
    cd ..
fi
cd ..

if [[ ''' + FORCE_BUILD + ''' == "true" ]]; then hasChanged=1; fi
if [[ ${hasChanged} -eq 1 ]]; then
  for QRP in ''' + QUAY_PROJECT + '''; do
    QUAY_REPO_PATH=""; if [[ ''' + PUSH_TO_QUAY + ''' == "true" ]]; then QUAY_REPO_PATH="${QRP}-rhel8"; fi
    curl \
"https://codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com/job/get-sources-rhpkg-container-build/buildWithParameters?\
token=CI_BUILD&\
cause=${QUAY_REPO_PATH}+respin+by+${BUILD_TAG}&\
GIT_BRANCH=''' + GIT_BRANCH + '''&\
GIT_PATHs=containers/codeready-workspaces-${QRP}&\
QUAY_REPO_PATHs=${QUAY_REPO_PATH}&\
JOB_BRANCH=master&\
FORCE_BUILD=true&\
SCRATCH=''' + SCRATCH + '''"
  done
fi

if [[ ${hasChanged} -eq 0 ]]; then
  echo "No changes upstream, nothing to commit"
fi
'''
    		}
	        def NEW_SHA = sh(script: '''#!/bin/bash -xe
	        cd ${WORKSPACE}/target; git rev-parse HEAD
	        ''', returnStdout: true)
	        println "Got NEW_SHA in target folder: " + NEW_SHA

	        if (NEW_SHA.equals(OLD_SHA) && !FORCE_BUILD.equals("true")) {
	          currentBuild.result='UNSTABLE'
	        }
    }
	}
}
