---
layout: section
title: "Beam Release Guide"
section_menu: section-menu/contribute.html
permalink: /contribute/release-guide/
---
<!--
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Apache Beam Release Guide

* TOC
{:toc}

## Introduction

The Apache Beam project periodically declares and publishes releases. A release is one or more packages of the project artifact(s) that are approved for general public distribution and use. They may come with various degrees of caveat regarding their perceived quality and potential for change, such as “alpha”, “beta”, “incubating”, “stable”, etc.

The Beam community treats releases with great importance. They are a public face of the project and most users interact with the project only through the releases. Releases are signed off by the entire Beam community in a public vote.

Each release is executed by a *Release Manager*, who is selected among the Beam committers. This document describes the process that the Release Manager follows to perform a release. Any changes to this process should be discussed and adopted on the [dev@ mailing list]({{ site.baseurl }}/get-started/support/).

Please remember that publishing software has legal consequences. This guide complements the foundation-wide [Product Release Policy](http://www.apache.org/dev/release.html) and [Release Distribution Policy](http://www.apache.org/dev/release-distribution).

## Overview

![Alt text]({{ "/images/release-guide-1.png" | prepend: site.baseurl }} "Release Process"){:width="100%"}

The release process consists of several steps:

1. Decide to release
1. Prepare for the release
1. Build a release candidate
1. Vote on the release candidate
1. During vote process, run validation tests
1. If necessary, fix any issues and go back to step 3.
1. Finalize the release
1. Promote the release

**********

## Decide to release

Deciding to release and selecting a Release Manager is the first step of the release process. This is a consensus-based decision of the entire community.

Anybody can propose a release on the dev@ mailing list, giving a solid argument and nominating a committer as the Release Manager (including themselves). There’s no formal process, no vote requirements, and no timing requirements. Any objections should be resolved by consensus before starting the release.

In general, the community prefers to have a rotating set of 3-5 Release Managers. Keeping a small core set of managers allows enough people to build expertise in this area and improve processes over time, without Release Managers needing to re-learn the processes for each release. That said, if you are a committer interested in serving the community in this way, please reach out to the community on the dev@ mailing list.

### Checklist to proceed to the next step

1. Community agrees to release
1. Community selects a Release Manager

**********

## Prepare for the release

Before your first release, you should perform one-time configuration steps. This will set up your security keys for signing the release and access to various release repositories.

To prepare for each release, you should audit the project status in the JIRA issue tracker, and do necessary bookkeeping. Finally, you should create a release branch from which individual release candidates will be built.

### One-time setup instructions

#### GPG Key

You need to have a GPG key to sign the release artifacts. Please be aware of the ASF-wide [release signing guidelines](https://www.apache.org/dev/release-signing.html). If you don’t have a GPG key associated with your Apache account, please create one according to the guidelines.

Get more entropy for creating a GPG key

    sudo apt-get install rng-tools
    sudo rngd -r /dev/urandom

Create a GPG key
    
    gpg --full-generate-key

Determine your Apache GPG Key and Key ID, as follows:

    gpg --list-keys

This will list your GPG keys. One of these should reflect your Apache account, for example:

    --------------------------------------------------
    pub   2048R/845E6689 2016-02-23
    uid                  Nomen Nescio <anonymous@apache.org>
    sub   2048R/BA4D50BE 2016-02-23

Here, the key ID is the 8-digit hex string in the `pub` line: `845E6689`.

Now, add your Apache GPG key to the Beam’s `KEYS` file both in [`dev`](https://dist.apache.org/repos/dist/dev/beam/KEYS) and [`release`](https://dist.apache.org/repos/dist/release/beam/KEYS) repositories at `dist.apache.org`. Follow the instructions listed at the top of these files. (Note: Only PMC members have write access to the release repository. If you end up getting 403 errors ask on the mailing list for assistance.)

Configure `git` to use this key when signing code by giving it your key ID, as follows:

    git config --global user.signingkey 845E6689

You may drop the `--global` option if you’d prefer to use this key for the current repository only.

You may wish to start `gpg-agent` to unlock your GPG key only once using your passphrase. Otherwise, you may need to enter this passphrase hundreds of times. The setup for `gpg-agent` varies based on operating system, but may be something like this:

    eval $(gpg-agent --daemon --no-grab --write-env-file $HOME/.gpg-agent-info)
    export GPG_TTY=$(tty)
    export GPG_AGENT_INFO

#### Access to Apache Nexus repository

Configure access to the [Apache Nexus repository](http://repository.apache.org/), which enables final deployment of releases to the Maven Central Repository.

1. You log in with your Apache account.
1. Confirm you have appropriate access by finding `org.apache.beam` under `Staging Profiles`.
1. Navigate to your `Profile` (top right dropdown menu of the page).
1. Choose `User Token` from the dropdown, then click `Access User Token`. Copy a snippet of the Maven XML configuration block.
1. Insert this snippet twice into your global Maven `settings.xml` file, typically `${HOME}/.m2/settings.xml`. The end result should look like this, where `TOKEN_NAME` and `TOKEN_PASSWORD` are your secret tokens:

        <settings>
          <servers>
            <server>
              <id>apache.releases.https</id>
              <username>TOKEN_NAME</username>
              <password>TOKEN_PASSWORD</password>
            </server>
            <server>
              <id>apache.snapshots.https</id>
              <username>TOKEN_NAME</username>
              <password>TOKEN_PASSWORD</password>
            </server>
          </servers>
        </settings>

#### Website development setup

Prepare to update the Beam website by following the [website development
instructions]({{ site.baseurl }}/contribute/website-contributions/).

#### Register to PyPI

Release manager needs to have an account with PyPI. If you need one, [register at PyPI](https://pypi.python.org/account/register/). You also need to be a maintainer (or an owner) of the [apache-beam](https://pypi.python.org/pypi/apache-beam) package in order to push a new release. Ask on the mailing list for assistance.

### Create a new version in JIRA

When contributors resolve an issue in JIRA, they are tagging it with a release that will contain their changes. With the release currently underway, new issues should be resolved against a subsequent future release. Therefore, you should create a release item for this subsequent release, as follows:

Attention: Only committer has permission to perform this.

1. In JIRA, navigate to the [`Beam > Administration > Versions`](https://issues.apache.org/jira/plugins/servlet/project-config/BEAM/versions).
1. Add a new release: choose the next minor version number compared to the one currently underway, select today’s date as the `Start Date`, and choose `Add`.

### Triage release-blocking issues in JIRA

There could be outstanding release-blocking issues, which should be triaged before proceeding to build a release candidate. We track them by assigning a specific `Fix version` field even before the issue resolved.

The list of release-blocking issues is available at the [version status page](https://issues.apache.org/jira/browse/BEAM/?selectedTab=com.atlassian.jira.jira-projects-plugin:versions-panel). Triage each unresolved issue with one of the following resolutions:

* If the issue has been resolved and JIRA was not updated, resolve it accordingly.
* If the issue has not been resolved and it is acceptable to defer this until the next release, update the `Fix Version` field to the new version you just created. Please consider discussing this with stakeholders and the dev@ mailing list, as appropriate.
* If the issue has not been resolved and it is not acceptable to release until it is fixed, the release cannot proceed. Instead, work with the Beam community to resolve the issue.

### Review Release Notes in JIRA

JIRA automatically generates Release Notes based on the `Fix Version` field applied to issues. Release Notes are intended for Beam users (not Beam committers/contributors). You should ensure that Release Notes are informative and useful.

Open the release notes from the [version status page](https://issues.apache.org/jira/browse/BEAM/?selectedTab=com.atlassian.jira.jira-projects-plugin:versions-panel) by choosing the release underway and clicking Release Notes.

You should verify that the issues listed automatically by JIRA are appropriate to appear in the Release Notes. Specifically, issues should:

* Be appropriately classified as `Bug`, `New Feature`, `Improvement`, etc.
* Represent noteworthy user-facing changes, such as new functionality, backward-incompatible API changes, or performance improvements.
* Have occurred since the previous release; an issue that was introduced and fixed between releases should not appear in the Release Notes.
* Have an issue title that makes sense when read on its own.

Adjust any of the above properties to the improve clarity and presentation of the Release Notes.

### Verify that a Release Build Works

Pre-installation for python build
* Install pip
  
  ```
  curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
  python get-pip.py
  ```
* Install virtualenv

  ```
  pip install --upgrade virtualenv
  ```
* Cython

  ```
  sudo pip install cython
  sudo apt-get install gcc
  sudo apt-get install python-dev
  ```
Make sure your ```time``` alias to ```/usr/bin/time```, if not:

```
sudo apt-get install time
alias time='/usr/bin/time'
```

Run gradle release build

* Clean current workspace

  ```
  git clean -fdx 
  ./gradlew clean
  ```

* Unlock the secret key
  ```
  gpg --output ~/doc.sig --sign ~/.bashrc
  ```

* Run build command
  ```
  ./gradlew build -PisRelease --no-parallel --scan --stacktrace
  ```
  
### Update and Verify Javadoc

The build with `-PisRelease` creates the combined Javadoc for the release in `sdks/java/javadoc`.

The file `sdks/java/javadoc/ant.xml` file contains a list of modules to include
in and exclude, plus a list of offline URLs that populate links from Beam's
Javadoc to the Javadoc for other modules that Beam depends on.

* Confirm that new modules added since the last release have been added to the
  inclusion list as appropriate.

* Confirm that the excluded package list is up to date.

* Verify the version numbers for offline links match the versions used by Beam. If
  the version number has changed, download a new version of the corresponding
  `<module>-docs/package-list` file.

### Create a release branch in apache/beam repository

Attention: Only committer has permission to create release branch in apache/beam.

Release candidates are built from a release branch. As a final step in preparation for the release, you should create the release branch, push it to the Apache code repository, and update version information on the original branch.

Check out the version of the codebase from which you start the release. For a new minor or major release, this may be `HEAD` of the `master` branch. To build a hotfix/incremental release, instead of the `master` branch, use the release tag of the release being patched. (Please make sure your cloned repository is up-to-date before starting.)

    git checkout <master branch OR release tag>

**NOTE**: If you are doing an incremental/hotfix release (e.g. 2.5.1), please check out the previous release tag, rather than the master branch.

Set up a few environment variables to simplify Maven commands that follow. (We use `bash` Unix syntax in this guide.)

    RELEASE=2.5.0
    NEXT_VERSION_IN_BASE_BRANCH=2.6.0
    BRANCH=release-${RELEASE}

Version represents the release currently underway, while next version specifies the anticipated next version to be released from that branch. Normally, 1.2.0 is followed by 1.3.0, while 1.2.3 is followed by 1.2.4.

**NOTE**: Only if you are doing an incremental/hotfix release (e.g. 2.5.1), please check out the previous release tag, before running the following instructions:

    BASE_RELEASE=2.5.0
    RELEASE=2.5.1
    NEXT_VERSION_IN_BASE_BRANCH=2.6.0
    git checkout tags/${BASE_RELEASE}

Create a new branch, and update version files in the master branch.

    git branch ${BRANCH}

    # Now change the version in existing gradle files, and Python files
    sed -i -e "s/'${RELEASE}'/'${NEXT_VERSION_IN_BASE_BRANCH}'/g" build_rules.gradle
    sed -i -e "s/${RELEASE}/${NEXT_VERSION_IN_BASE_BRANCH}/g" gradle.properties
    sed -i -e "s/${RELEASE}/${NEXT_VERSION_IN_BASE_BRANCH}/g" sdks/python/apache_beam/version.py

    # Save changes in master branch
    git add gradle.properties build_rules.gradle sdks/python/apache_beam/version.py
    git commit -m "Moving to ${NEXT_VERSION_IN_BASE_BRANCH}-SNAPSHOT on master branch."

Check out the release branch.

    git checkout ${BRANCH}
    
The rest of this guide assumes that commands are run in the root of a repository on `${BRANCH}` with the above environment variables set.

### Update the Python SDK version

Update [sdks/python/apache_beam/version.py](https://github.com/apache/beam/blob/master/sdks/python/apache_beam/version.py) in both master branch and release branch.

* In the release branch, update the Python SDK version to the release version(e.g. `2.5.0-dev` to `2.5.0`).

### Update release specific configurations

Update Java runner specific configurations in release branch:
* [beam/runners/google-cloud-dataflow-java/build.gradle](https://github.com/apache/beam/blob/master/runners/google-cloud-dataflow-java/build.gradle): change value of 'dataflow.container_version' to 'beam-release_version_number'(e.g, 'beam-master-20180601' to 'beam-2.5.0')
* [beam/runners/google-cloud-dataflow-java/pom.xml](https://github.com/apache/beam/blob/master/runners/google-cloud-dataflow-java/pom.xml): change value of 'version' field (e.g, '2.5.0-SNAPSHOT' to '2.5.0') 

### Start a snapshot build

Start a build of [the nightly snapshot](https://builds.apache.org/view/A-D/view/Beam/job/beam_Release_NightlySnapshot/) against master branch.
Some processes, including our archetype tests, rely on having a live SNAPSHOT of the current version
from the `master` branch. Once the release branch is cut, these SNAPSHOT versions are no longer found,
so builds will be broken until a new snapshot is available.

Comment  ```Run Gradle Publish``` in one pull request to trigger build.

### Checklist to proceed to the next step

1. Release Manager’s GPG key is published to `dist.apache.org`
2. Release Manager’s GPG key is configured in `git` configuration
3. Release Manager has `org.apache.beam` listed under `Staging Profiles` in Nexus
4. Release Manager’s Nexus User Token is configured in `settings.xml`
5. JIRA release item for the subsequent release has been created
6. There are no release blocking JIRA issues
7. Release Notes in JIRA have been audited and adjusted
8. Combined javadoc has the appropriate contents.
9. Release branch has been created
10. Originating branch has the version information updated to the new version
11. Nightly snapshot is in progress (do revisit it continually)

**********

## Build a release candidate

The core of the release process is the build-vote-fix cycle. Each cycle produces one release candidate. The Release Manager repeats this cycle until the community approves one release candidate, which is then finalized.

### Build and stage Java artifacts with Gradle

Set up a few environment variables to simplify the commands that follow. These identify the release candidate being built, and the branch where you will stage files. Start with `RC_NUM` equal to `1` and increment it for each candidate.

    RC_NUM=1

Make sure your git config will maintain your account:
    
    git config credential.helper store

Use Gradle release plugin to build the release artifacts, and push code and
release tag to the origin repository (this would be the Apache Beam repo):

    ./gradlew release -Prelease.newVersion=${RELEASE}-SNAPSHOT \
                  -Prelease.releaseVersion=${RELEASE}-RC${RC_NUM} \
                  -Prelease.useAutomaticVersion=true --info --no-daemon

Use Gradle publish plugin to stage these artifacts on the Apache Nexus repository, as follows:

    ./gradlew publish -PisRelease --no-parallel --no-daemon

Review all staged artifacts. They should contain all relevant parts for each module, including `pom.xml`, jar, test jar, javadoc, etc. Artifact names should follow [the existing format](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.apache.beam%22) in which artifact name mirrors directory structure, e.g., `beam-sdks-java-io-kafka`. Carefully review any new artifacts.

Close the staging repository on Apache Nexus. When prompted for a description, enter “Apache Beam, version X, release candidate Y”.

### Stage source release on dist.apache.org

Attention: Only committer has permissions to perform following steps.

Copy the source release to the dev repository of `dist.apache.org`.

1. If you have not already, check out the Beam section of the `dev` repository on `dist.apache.org` via Subversion. In a fresh directory:

        svn co https://dist.apache.org/repos/dist/dev/beam

1. Make a directory for the new release:

        mkdir beam/${RELEASE}
        cd beam/${RELEASE}

1. Download source zip from GitHub:

    wget https://github.com/apache/beam/archive/release-${RELEASE}.zip \
         -O apache-beam-${RELEASE}-source-release.zip

1. Create hashes and sign the source distribution:

        gpg --armor --detach-sig apache-beam-${RELEASE}-source-release.zip
        sha512sum apache-beam-${RELEASE}-source-release.zip > apache-beam-${RELEASE}-source-release.zip.sha512

1. Add and commit all the files.

        svn add beam/${RELEASE}
        svn commit

1. Verify that files are [present](https://dist.apache.org/repos/dist/dev/beam).

### Stage python binaries on dist.apache.org

Build python binaries in release branch in sdks/python dir.

    python setup.py sdist --format=zip
    cd dist
    cp apache-beam-${RELEASE}.zip staging/apache-beam-${RELEASE}-python.zip
    cd staging

Create hashes and sign the binaries

    gpg --armor --detach-sig apache-beam-${RELEASE}-python.zip
    sha512sum apache-beam-${RELEASE}-python.zip > apache-beam-${RELEASE}-python.zip.sha512

Staging binaries

    svn co https://dist.apache.org/repos/dist/dev/beam
    cd beam/${RELEASE}
    svn add *
    svn commit

Verify that files are [present](https://dist.apache.org/repos/dist/dev/beam).

### Build the Pydoc API reference

Make sure you have ```tox``` installed: 

```
pip install tox

```
Create the Python SDK documentation using sphinx by running a helper script.
```
cd sdks/python && tox -e docs
```
By default the Pydoc is generated in `sdks/python/target/docs/_build`. Let `${PYDOC_ROOT}` be the absolute path to `_build`.

### Propose a pull request for website updates

The final step of building the candidate is to propose a website pull request.

Start by updating `release_latest` version flag in the top-level `_config.yml`, and list the new release in the [Apache Beam Downloads]({{ site.baseurl }}/get-started/downloads/), linking to the source code download and the Release Notes in JIRA.

Beam publishes API reference manual for each release on the website. For Java SDK, that’s Javadoc.

One of the artifacts created in the release contains the Javadoc for the
website. To update the website, you must unpack this jar file from the release
candidate into the source tree of the website.

Add the new Javadoc to [SDK API Reference page]({{ site.baseurl }}/documentation/sdks/javadoc/) page, as follows:

* Unpack the Maven artifact `org.apache.beam:beam-sdks-java-javadoc` into some temporary location. Call this `${JAVADOC_TMP}`.
* Copy the generated Javadoc into the website repository: `cp -r ${JAVADOC_TMP} src/documentation/sdks/javadoc/${RELEASE}`.
* Set up the necessary git commands to account for the new and deleted files from the javadoc.
* Update the Javadoc link on this page to point to the new version (in `src/documentation/sdks/javadoc/current.md`).

#### Create Pydoc
Add the new Pydoc to [SDK API Reference page]({{ site.baseurl }}/documentation/sdks/pydoc/) page, as follows:

* Copy the generated Pydoc into the website repository: `cp -r ${PYDOC_ROOT} src/documentation/sdks/pydoc/${RELEASE}`.
* Remove `.doctrees` directory.
* Update the Pydoc link on this page to point to the new version (in `src/documentation/sdks/pydoc/current.md`).

Finally, propose a pull request with these changes. (Don’t merge before finalizing the release.)

### Checklist to proceed to the next step

1. Maven artifacts deployed to the staging repository of [repository.apache.org](https://repository.apache.org/content/repositories/)
1. Source distribution deployed to the dev repository of [dist.apache.org](https://dist.apache.org/repos/dist/dev/beam/)
1. Website pull request proposed to list the [release]({{ site.baseurl }}/get-started/downloads/), publish the [Java API reference manual]({{ site.baseurl }}/documentation/sdks/javadoc/), and publish the [Python API reference manual]({{ site.baseurl }}/documentation/sdks/pydoc/).

You can (optionally) also do additional verification by:
1. Check that Python zip file contains the `README.md`, `NOTICE`, and `LICENSE` files.
1. Check hashes (e.g. `md5sum -c *.md5` and `sha1sum -c *.sha1`)
1. Check signatures (e.g. `gpg --verify apache-beam-1.2.3-python.zip.asc apache-beam-1.2.3-python.zip`)
1. `grep` for legal headers in each file.
1. Run all jenkins suites and include links to passing tests in the voting email. (Select "Run with parameters")

**********

## Vote on the release candidate

Once you have built and individually reviewed the release candidate, please share it for the community-wide review. Please review foundation-wide [voting guidelines](http://www.apache.org/foundation/voting.html) for more information.

Start the review-and-vote thread on the dev@ mailing list. Here’s an email template; please adjust as you see fit.

    From: Release Manager
    To: dev@beam.apache.org
    Subject: [VOTE] Release 1.2.3, release candidate #3

    Hi everyone,
    Please review and vote on the release candidate #3 for the version 1.2.3, as follows:
    [ ] +1, Approve the release
    [ ] -1, Do not approve the release (please provide specific comments)


    The complete staging area is available for your review, which includes:
    * JIRA release notes [1],
    * the official Apache source release to be deployed to dist.apache.org [2], which is signed with the key with fingerprint FFFFFFFF [3],
    * all artifacts to be deployed to the Maven Central Repository [4],
    * source code tag "v1.2.3-RC3" [5],
    * website pull request listing the release and publishing the API reference manual [6].
    * Java artifacts were built with Maven MAVEN_VERSION and OpenJDK/Oracle JDK JDK_VERSION.
    * Python artifacts are deployed along with the source release to the dist.apache.org [2].

    The vote will be open for at least 72 hours. It is adopted by majority approval, with at least 3 PMC affirmative votes.

    Thanks,
    Release Manager

    [1] link
    [2] link
    [3] https://dist.apache.org/repos/dist/release/beam/KEYS
    [4] link
    [5] link
    [6] link

If there are any issues found in the release candidate, reply on the vote thread to cancel the vote. There’s no need to wait 72 hours. Proceed to the `Fix Issues` step below and address the problem. However, some issues don’t require cancellation. For example, if an issue is found in the website pull request, just correct it on the spot and the vote can continue as-is.

If there are no issues, reply on the vote thread to close the voting. Then, tally the votes in a separate email. Here’s an email template; please adjust as you see fit.

    From: Release Manager
    To: dev@beam.apache.org
    Subject: [RESULT] [VOTE] Release 1.2.3, release candidate #3

    I'm happy to announce that we have unanimously approved this release.

    There are XXX approving votes, XXX of which are binding:
    * approver 1
    * approver 2
    * approver 3
    * approver 4

    There are no disapproving votes.

    Thanks everyone!
    
### Run validation tests
All tests listed in this [spreadsheet](https://s.apache.org/beam-release-validation)

_Note_: -Prepourl and -Pver can be found in the RC vote email sent by Release Manager.

* Java Quickstart Validation

  Direct Runner:
  ```
  ./gradlew :beam-runners-direct-java:runQuickstartJavaDirect \ 
  -Prepourl=https://repository.apache.org/content/repositories/orgapachebeam-${KEY} \
  -Pver=${RELEASE_VERSION}
  ```
  Apex Local Runner
  ```
  ./gradlew :beam-runners-apex:runQuickstartJavaApex \
  -Prepourl=https://repository.apache.org/content/repositories/orgapachebeam${KEY} \
  -Pver=${RELEASE_VERSION}
  ```
  Flink Local Runner
  ```
  ./gradlew :beam-runners-flink_2.11:runQuickstartJavaFlinkLocal \
  -Prepourl=https://repository.apache.org/content/repositories/orgapachebeam-${KEY} \
  -Pver=${RELEASE_VERSION}
  ```
  Spark Local Runner
  ```
  ./gradlew :beam-runners-spark:runQuickstartJavaSpark \
  -Prepourl=https://repository.apache.org/content/repositories/orgapachebeam-${KEY} \
  -Pver=${RELEASE_VERSION}
  ```
  Dataflow Runner
  ```
  ./gradlew :beam-runners-google-cloud-dataflow-java:runQuickstartJavaDataflow \
  -Prepourl=https://repository.apache.org/content/repositories/orgapachebeam-${KEY} \
  -Pver= ${RELEASE_VERSION}\
  -PgcpProject=${YOUR_GCP_PROJECT} \
  -PgcsBucket=${YOUR_GCP_BUCKET}
  ```
* Java Mobile Game(UserScore, HourlyTeamScore, Leaderboard)
  
  Pre-request
  * Create your own BigQuery dataset
    ```
    bq mk --project=${YOUR_GCP_PROJECT} ${YOUR_DATASET}
    ```
  * Create yout PubSub topic 
    ```
    gcloud alpha pubsub topics create --project=${YOUR_GCP_PROJECT} ${YOUR_PROJECT_PUBSUB_TOPIC} 
    ```
  * Setup your service account
    
    Goto IAM console in your project to create a service account as ```project owner```
    
    Run 
    ```
    gcloud iam service-accounts keys create ${YOUR_KEY_JSON} --iam-account ${YOUR_SERVICE_ACCOUNT_NAME}@${YOUR_PROJECT_NAME}
    export GOOGLE_APPLICATION_CREDENTIALS=${PATH_TO_YOUR_KEY_JSON} 
    ```
  Run 
  ```
  ./gradlew :beam-runners-google-cloud-dataflow-java:runMobileGamingJavaDataflow \
   -Prepourl=https://repository.apache.org/content/repositories/orgapachebeam-${KEY} \ 
   -Pver= ${RELEASE_VERSION}\
   -PgcpProject=${YOUR_GCP_PROJECT} \
   -PgcsBucket=gs://dataflow-samples/game/gaming_data1.csv \
   -PbqDataset=${YOUR_DATASET} -PpubsubTopic=${YOUR_PROJECT_PUBSUB_TOPIC}
  ```
* Python Quickstart(batch & streaming), MobileGame(UserScore, HourlyTeamScore)
  
  Create a new PR in apache/beam
  
  In comment area, type in ```Run Python ReleaseCandidate```
 
* Python Leaderboard & GameStats
  * Get staging RC ```wget https://dist.apache.org/repos/dist/dev/beam/2.5.0/* ```
  * Verify the hashes
  
    ```
    sha512sum -c apache-beam-2.5.0-python.zip.sha512
    sha512sum -c apache-beam-2.5.0-source-release.zip.sha512
    ```
  * Build SDK
  
    ```
    sudo apt-get install unzip
    unzip apache-beam-2.5.0-source-release.zip
    python setup.py sdist
    ```
  * Setup virtualenv
    
    ```
    pip install --upgrade pip
    pip install --upgrade setuptools
    pip install --upgrade virtualenv
    virtualenv beam_env
     . beam_env/bin/activate
    ```
  * Install SDK
    
    ```
    pip install dist/apache-beam-2.5.0.tar.gz
    pip install dist/apache-beam-2.5.0.tar.gz[gcp]
    ```
  * Setup GCP
    
    Please repeat following steps for every following test.
    
    ```
    bq rm -rf --project=${YOUR_PROJECT} ${USER}_test
    bq mk --project=${YOUR_PROJECT} ${USER}_test
    gsutil rm -rf ${YOUR_GS_STORAGE]
    gsutil mb -p ${YOUR_PROJECT} ${YOUR_GS_STORAGE}
    gcloud alpha pubsub topics create --project=${YOUR_PROJECT} ${YOUR_PUBSUB_TOPIC}
    ```
    Setup your service account as described in ```Java Mobile Game``` section above.
  
    Produce data by using java injector:
    * Configure your ~/.m2/settings.xml as following:
      ```
      <settings>
        <profiles>
          <profile>
            <id>release-repo</id>
            <activation>
              <activeByDefault>true</activeByDefault>
            </activation>
            <repositories>
              <repository>
                <id>Release 2.4.0 RC3</id>
                <name>Release 2.4.0 RC3</name>
                <url>https://repository.apache.org/content/repositories/orgapachebeam-1031/</url>
              </repository>
            </repositories>
          </profile>
        </profiles>
      </settings>
      ```
      _Note_: You can found the latest  ```id```, ```name``` and ```url``` for one RC in the vote email thread sent out by Release Manager.
    
    * Run 
      ```
      mvn archetype:generate \
            -DarchetypeGroupId=org.apache.beam \
            -DarchetypeArtifactId=beam-sdks-java-maven-archetypes-examples \
            -DarchetypeVersion=${RELEASE_VERSION} \
            -DgroupId=org.example \
            -DartifactId=word-count-beam \
            -Dversion="0.1" \
            -Dpackage=org.apache.beam.examples \
            -DinteractiveMode=false
            -DarchetypeCatalog=internal
            
      mvn compile exec:java -Dexec.mainClass=org.apache.beam.examples.complete.game.injector.Injector \
        -Dexec.args="${YOUR_PROJECT} ${YOUR_PUBSUB_TOPIC} none"
      ```
  * Run Leaderboard with Direct Runner
    ```
    python -m apache_beam.examples.complete.game.leader_board \
    --project=${YOUR_PROJECT} \
    --topic projects/${YOUR_PROJECT}/topics/${YOUR_PUBSUB_TOPIC} \
    --dataset ${USER}_test
    ```
    Inspect results:
    * Check whether there is any error messages in console.
    * Goto your BigQuery console and check whether your ${USER}_test has leader_board_users and leader_board_teams table.
    * bq head -n 10 ${USER}_test.leader_board_users
    * bq head -n 10 ${USER}_test.leader_board_teams
  * Run Leaderboard with Dataflow Runner
    ```
    python -m apache_beam.examples.complete.game.leader_board \ 
    --project=${YOUR_PROJECT} \ 
    --topic projects/${YOUR_PROJECT}/topics/${YOUR_PUBSUB_TOPIC} \ 
    --dataset ${USER}_test \ 
    --runner DataflowRunner \ 
    --temp_location=${YOUR_GS_BUCKET}/temp/ \ 
    --sdk_location dist/*
    ```
    Inspect results:
    * Goto your Dataflow job console and check whether there is any error.
    * Goto your BigQuery console and check whether your ${USER}_test has leader_board_users and leader_board_teams table.
    * bq head -n 10 ${USER}_test.leader_board_users
    * bq head -n 10 ${USER}_test.leader_board_teams
  * Run GameStats with Direct Runner
    ```
    python -m apache_beam.examples.complete.game.game_stats \
    --project=${YOUR_PROJECT} \
    --topic projects/${YOUR_PROJECT}/topics/${YOUR_PUBSUB_TOPIC} \
    --dataset ${USER}_test \
    --fixed_window_duration ${SOME_SMALL_DURATION}
    ```
    Inspect results:
    * Check whether there is any error messages in console.
    * Goto your BigQuery console and check whether your ${USER}_test has game_stats_teams and game_stats_sessions table.
    * bq head -n 10 ${USER}_test.game_stats_teams
    * bq head -n 10 ${USER}_test.game_stats_sessions

  * Run GameStats with Dataflow Runner
    ```
    python -m apache_beam.examples.complete.game.game_stats \ 
    --project=${YOUR_PROJECT} \ 
    --topic projects/${YOUR_PROJECT}/topics/${YOUR_PUBSUB_TOPIC} \ 
    --dataset ${USER}_test \ 
    --runner DataflowRunner \ 
    --temp_location=${YOUR_GS_BUCKET}/temp/ \ 
    --sdk_location dist/* \
    --fixed_window_duration ${SOME_SMALL_DURATION}
    ```
    Inspect results:
    * Goto your Dataflow job console and check whether there is any error.
    * Goto your BigQuery console and check whether your ${USER}_test has game_stats_teams and game_stats_sessions table.
    * bq head -n 10 ${USER}_test.game_stats_teams
    * bq head -n 10 ${USER}_test.game_stats_sessions
### Checklist to proceed to the finalization step

1. Community votes to release the proposed candidate, with at least three approving PMC votes

**********

## Fix any issues

Any issues identified during the community review and vote should be fixed in this step.

Code changes should be proposed as standard pull requests to the `master` branch and reviewed using the normal contributing process. Then, relevant changes should be cherry-picked into the release branch. The cherry-pick commits should then be proposed as the pull requests against the release branch, again reviewed and merged using the normal contributing process.

Once all issues have been resolved, you should go back and build a new release candidate with these changes.

### Checklist to proceed to the next step

1. Issues identified during vote have been resolved, with fixes committed to the release branch.

**********

## Finalize the release

Once the release candidate has been reviewed and approved by the community, the release should be finalized. This involves the final deployment of the release candidate to the release repositories, merging of the website changes, etc.

### Deploy artifacts to Maven Central Repository

Use the Apache Nexus repository to release the staged binary artifacts to the Maven Central repository. In the `Staging Repositories` section, find the relevant release candidate `orgapachebeam-XXX` entry and click `Release`. Drop all other release candidates that are not being released.

### Deploy Python artifacts to PyPI

1. Create a new release and upload the Python zip file for the new release using the [PyPI UI] (https://pypi.python.org/pypi/apache-beam)
1. Alternatively, use the command line tool to upload the new release `twine upload apache-beam-${RELEASE}.zip`

Note: It is important to rename `apache-beam-${RELEASE}-python.zip` to `apache-beam-${RELEASE}.zip` before uploading, because PyPI expects a filename in the `<package-name>-<package-version>` format.

#### Deploy source release to dist.apache.org

Copy the source release from the `dev` repository to the `release` repository at `dist.apache.org` using Subversion.

### Git tag

Create and push a new signed tag for the released version by copying the tag for the final release candidate, as follows:

    VERSION_TAG="v${RELEASE}"
    git tag -s "$VERSION_TAG" "$RC_TAG"
    git push github "$VERSION_TAG"

### Merge website pull request

Merge the website pull request to [list the release]({{ site.baseurl }}/get-started/downloads/), publish the [Python API reference manual]({{ site.baseurl }}/documentation/sdks/pydoc/), and the [Java API reference manual]({{ site.baseurl }}/documentation/sdks/javadoc/) created earlier.

### Mark the version as released in JIRA

In JIRA, inside [version management](https://issues.apache.org/jira/plugins/servlet/project-config/BEAM/versions), hover over the current release and a settings menu will appear. Click `Release`, and select today’s date.

### Recordkeeping with ASF

Use reporter.apache.org to seed the information about the release into future project reports.

### Checklist to proceed to the next step

* Maven artifacts released and indexed in the [Maven Central Repository](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.apache.beam%22)
* Source distribution available in the release repository of [dist.apache.org](https://dist.apache.org/repos/dist/release/beam/)
* Source distribution removed from the dev repository of [dist.apache.org](https://dist.apache.org/repos/dist/dev/beam/)
* Website pull request to [list the release]({{ site.baseurl }}/get-started/downloads/) and publish the [API reference manual]({{ site.baseurl }}/documentation/sdks/javadoc/) merged
* Release tagged in the source code repository
* Release version finalized in JIRA. (Note: Not all committers have administrator access to JIRA. If you end up getting permissions errors ask on the mailing list for assistance.)
* Release version is listed at reporter.apache.org

**********

## Promote the release

Once the release has been finalized, the last step of the process is to promote the release within the project and beyond.

### Apache mailing lists

Announce on the dev@ mailing list that the release has been finished.

Announce on the release on the user@ mailing list, listing major improvements and contributions.

Announce the release on the announce@apache.org mailing list.

### Beam blog

Major or otherwise important releases should have a blog post. Write one if needed for this particular release. Minor releases that don’t introduce new major functionality don’t necessarily need to be blogged.

### Social media

Tweet, post on Facebook, LinkedIn, and other platforms. Ask other contributors to do the same.

Also, update [the Wikipedia article on Apache Beam](https://en.wikipedia.org/wiki/Apache_Beam).

### Checklist to declare the process completed

1. Release announced on the user@ mailing list.
1. Blog post published, if applicable.
1. Release recorded in reporter.apache.org.
1. Release announced on social media.
1. Completion declared on the dev@ mailing list.
1. Update Wikipedia Apache Beam article.

**********

## Improve the process

It is important that we improve the release processes over time. Once you’ve finished the release, please take a step back and look what areas of this process and be improved. Perhaps some part of the process can be simplified. Perhaps parts of this guide can be clarified.

If we have specific ideas, please start a discussion on the dev@ mailing list and/or propose a pull request to update this guide. Thanks!
