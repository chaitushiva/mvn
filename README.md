# mvn
MVN CHEATSHEET
Maven Cheat Sheet
=================

My collection of useful hints and snippets for [Apache Maven](http://maven.apache.org/).

POM - Project Object Model
--------------------------

See [http://maven.apache.org/pom.html](http://maven.apache.org/pom.html)

### Minimal POM

````xml
<project>
     <modelVersion>4.0.0</modelVersion>
     <groupId>com.packt</groupId>
     <artifactId>sample-one</artifactId>
     <version>1.0.0</version>
</project>
````

### Super POM

Bundled with Maven in `MAVEN_HOME/lib/maven-model-builder- 3.2.3.jar - org/apache/maven/model/pom-4.0.0.xml` - defines

* Maven central repository
* Maven central plugin repository
* Required information to build a project in `<build>`
* A `<reporting>` section
* The default build profile

### Overriding Values in POM

Define "blocks" with the same ID as in the Super POM. Show "effective" POM with

````shell
mvn help:effective-pom
````

### Maven Coordinates

* Identify a project, dependency or plugin
* A combination of `groupId`, `artifact` and `version`
* Plugins don't need a version, as a default, `org.apache.maven.plugins` or `org. codehaus.mojo` is used

### Parent POM Files

* Used to **avoid redundancy**

Version Handling
----------------

Set version in `<project><version>...</version></project>` from the command line with

    mvn versions:set versions:commit -DnewVersion=1.0.0-SNAPSHOT


Maven Release Plugin
--------------------

The [Maven Release Plugin](http://maven.apache.org/maven-release/maven-release-plugin/) let's you upload artifacts to Maven repositories - e.g. Artifactory.

### Jenkins Pipeline for Uploading Maven Artifacts

````groovy
stage 'Clone from Git'
node {
    checkout scm
}

stage 'Build'
node {
    sh 'mvn clean install -DskipTests'
}

stage 'Test'
node {
    sh 'mvn test'
}


stage 'Upload to Artifactory'
timeout(time:5, unit:'MINUTES') {
    input message: 'Do you want to push artifacts to Artifactory?'

    node {
        def pom = readMavenPom file: 'pom.xml'
        def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")

        /**
         * Clean any locally modified files and ensure we are actually on origin/master
         * as a failed release could leave the local workspace ahead of origin/master
         */
        sh "git clean -f && git reset --hard origin/master"

        /**
         * Description of parameters (for further details, see http://maven.apache.org/maven-release/maven-release-plugin/)
         *
         *    -DreleaseVersion            Default version to use when preparing a release or a branch.
         *    -DdevelopmentVersion        Default version to use for new local working copy
         *    -DpushChanges               Implemented with git will or not push changes to the upstream repository
         *    -DlocalCheckout             Use a local checkout instead of doing a checkout from the upstream repository
         *    -DpreparationGoals          Goals to run as part of the preparation step, after transformation but before committing
         *    -Darguments="-DskipTests"   Skips the tests in the release goal - see http://stackoverflow.com/questions/8685100/how-can-i-get-maven-release-plugin-to-skip-my-tests
         *    -B                          Run in non-interactive (batch) mode
         *
         *  Preparation steps:
         *
         *    release:prepare             see http://maven.apache.org/maven-release/maven-release-plugin/prepare-mojo.html
         *    release:perform             see http://maven.apache.org/maven-release/maven-release-plugin/perform-mojo.html
         */
        sh """mvn \
            -DreleaseVersion=${version} \
            -DdevelopmentVersion=${pom.version} \
            -DpushChanges=false \
            -DlocalCheckout=true \
            -DpreparationGoals=initialize \
            -Darguments="-DskipTests" \
            release:prepare release:perform \
            -B"""

        sh "git push origin ${pom.artifactId}-${version}"
    }

}
````

Artifactory Release Management
------------------------------

see [https://wiki.jenkins-ci.org/display/JENKINS/Jenkins+Artifactory+Plugin+-+Release+Management](https://wiki.jenkins-ci.org/display/JENKINS/Jenkins+Artifactory+Plugin+-+Release+Management)


Troubleshooting Maven
=====================

Problems with Maven Release Plugin
----------------------------------

* Update your Git version
* Make sure to have the most recent version of the plugin set in your `pom.xml`

    ```XML
    <plugin>
           <groupId>org.apache.maven.plugins</groupId>
           <artifactId>maven-release-plugin</artifactId>
           <version>2.5.3</version>
  </plugin>
    ```

Maven does not upload RELEASE version but SNAPSHOT
-------------------------------------------------

See [https://issues.apache.org/jira/browse/MRELEASE-812](https://issues.apache.org/jira/browse/MRELEASE-812).


Further Resources
=================

* [Book: Mastering Apache Maven 3](http://shop.oreilly.com/product/9781783983865.do)
* [DZone: Why I never use the Maven release plugin](https://dzone.com/articles/why-i-never-use-maven-release)
* [Maven Release Plugin: The Final Nail in the Coffin](https://axelfontaine.com/blog/final-nail.html)
* [Maven Cheat Sheet](https://confluence.sakaiproject.org/display/REL/Maven+release+plugin+cheat+sheet)
* [About publishing Maven Artifacts](http://www.apache.org/dev/publishing-maven-artifacts.html#publish-snapshot)
