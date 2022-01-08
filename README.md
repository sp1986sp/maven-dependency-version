# Maven dependency version control for multiple microservices

With the start of OneShop, we always tried to make things simpler for internal management. OneShop platform currently has 18+ microservices that are deployed over AWS EKS K8. As we are building this product for multiple countries, there are always many environments to deal with. These countries are generally on different versions of the microservices.
To deal with that many numbers of microservices on different versions it was always a challenge to have shared and common dependencies jars like platform, hal-adapter build first with the precise version required for the deployment of microservices.


#### Now let's move to the challenge we faced during deployment

For each deployment of microservice the dependency jars needed to be built prior and for that there were dependencies Jenkins jobs that need to be triggered so that the required jars are present in the local maven repository of machines before the deployment of microservice. This causes the below issues:
* The step of building this dependencies jar was an overhead before the deployment of any microservice.
* The dependencies are not released over nexus through the Jenkins pipeline
* To get the older jar version we have to build the dependency repository again.
* Dependencies must be fetched from Nexus for all the environments including local.

#### To overcome the above-mentioned problems, we built the below solution based on Maven
We decided to publish the dependencies jars to the nexus repository. The only catch was that we require to maintain the dependencies version with each release. For that we decided to update the version with every sprint and the nomenclature used for naming the branch was release-44, release-45 for sprint-44, and sprint-45 respectively. With every release, the jar version like 1.44.0-SNAPSHOT and 1.45.0-SNAPSHOT for release-44 and release-45 branch respectively is published to Nexus repository. In this way, we can have a dependencies jar of each release present in the Nexus repository and the one that is required for the microservice could be fetched accordingly.

Now, we met another hurdle of updating the dependencies jar version in each microservice( Remember there are 18 of them ) with every release. That was a huge effort and so to mitigate this we came up with a solution of creating a parent-pom file. This parent-pom file was the parent of all the microservices and dependencies pom. And all the dependencies versions info was moved to this parent-pom file. This reduced the update of the versions of dependencies to a single file update and that will resolve the dependencies in each service. This parent-pom file was then published to Nexus repo in a separate folder with the nomenclature: 1_44_0, 1_45_0 which corresponds to Sprint-44 and Sprint-45 respectively. The version of parent pom will always remain the same as it only contains the jar version and dependency management.

Sample parent pom file: https://github.com/sp1986sp/maven-dependency-version/blob/main/pom.xml

Note with this change following were the rules that were required to be followed:
* All jars and parent pom must have a SNAPSHOT version as the snapshot version can be replaced locally on running: **mvn clean install -U**
* The version of parent pom will always remain the same as it only contains the jar version and dependency management.
* The Nexus folder in which the parent-pom will be published will change according to the sprint version. And the nomenclature of the folder will be like 1_37_0, 1_38_0, 1_39_0
* Dependencies will always be published from Jenkins's job only. Local publish will not be allowed.
* The jar version of the platform, hal-adapter should be the same as the folder in which the parent-pom is published. For example, let say we want to publish a 1.44.0-SNAPSHOT version for all jars then the parent-pom ( always having a version of itself as an example: 1.2.0-SNAPSHOT ) published in the 1_44_0 folder.

With the introduction of parent-pom and movement of versions information there, we need to come up with a way to release the dependencies. The solution was to release the dependencies in three phases:
* First Phase: **dev**
* Second Phase: **qa**
* Final Phase: **release**

These three phases were created as profiles in the parent-pom file ( as you can see in the snippet of the parent-pom file). These profiles are used while publishing the dependencies using Jenkins jobs.

#### Dependency Publish using Jenkins Job
In this solution, we created 4 nexus repositories:
1) eshop-dev
2) eshop-qa
3) eshop-release
4) eshop-common

In this solution we created 4 new Jenkins jobs:
* One for publishing all dependencies jars to dev nexus repo: **DEV Jenkins job**
* One for publishing jars to qa nexus repo: **QA Jenkins job**
* One for publishing jars to release nexus repo: **RELEASE Jenkins job**
* One for publishing parent-pom in parent-pom nexus repo ( the parent-pom will be used from a single location for all the environments ): **PARENT-POM Jenkins job**

The same jar version will be used from different nexus repo based on the profile passed during building and starting of service, like for dev env to provide profile:

 `mvn clean install -P eshop-dev -Dversion=1_44_0`

This command will download the dependencies from **eshop-dev** and the version of jars will be **1.44.0-SNAPSHOT** as defined in the parent pom. For example, platform: 1.44.0-SNAPSHOT will be downloaded from the **eshop-dev** nexus repo if the profile passed is **eshop-dev** and the same jar version will be downloaded from the **eshop-qa** nexus repository if the profile passed is **eshop-qa**.

In this way the dependencies jar follow these steps to reach production:
1) Development changes are done and published to the **eshop-dev** nexus repository
2) Once the development is complete the jar is published to the **eshop-qa** nexus repository.
3) If any bug is raised the changes on dependencies are again done and published on **eshop-dev** and once verified then is again pushed to **eshop-qa** for further testing.
4) Finally, at the end of the sprint, the final verified jar is now published to the **eshop-release** nexus repository. This repository is used by production and Natco connected environments.

The jar version is defined only in the parent pom file. In all the service repositories the version defined in parent-pom will be inherited for all dependencies jar versions.

#### Steps to setup maven local repository and download the dependencies jar
We will define two repositories ( this is for local development as well ):
1) **remote**: It should always fetch all dependencies from the Nexus repo. Dependencies like platform, hal-adapter, etc. should not be locally built in this repo. This repository must always download the dependencies ( Because on the local building of a dependency a maven-metadata-local.xml is generated in a local path where the jar is generated, this file does not allow the download from Nexus repo.)

    * **The path on the local system will be: .m2/remote**

2. **local**: It should only fetch parent-pom from Nexus repo rest all dependencies must be locally built to be used. Every time we want to change any dependency-like the platform for let say 1.44.0-SNAPSHOT version we need to do changes locally and then build a platform using this local repo.
   
    * **The path on the local system will be: .m2/local**

#### Define the maven settings.xml
We have a **common** profile having all common repositories ( parent-pom, maven central, plugins, etc ) defined and it is by **default** activated in settings.xml. There are three more profiles defined for fetching dependencies from the 3 different nexus repo: **eshop-dev**, **eshop-qa**, and **eshop-release**. Changes in settings.xml to support **common**, **eshop-dev**, **eshop-qa**, and **eshop-release** profile.

Sample maven settings.xml file: https://github.com/sp1986sp/maven-dependency-version/blob/main/settings.xml
  
#### Local Repository Development on Local Environment
Let us assume we need to develop some feature that has changes in dependencies as well as in the service repository. So, we need to do the changes in dependencies on local and need to verify if the changes are compatible with the service repository :
1.  First, do the changes in the dependencies repository and build it in the local repo using this command:

    `mvn -Dmaven.repo.local=$HOME/.m2/local -Dversion=1_44_0 clean install`
    
    **-Dmaven.repo.local=$HOME/.m2/local:** It tells that we need to use local repo in the local system.
    
    **-Dversion:** Nexus Folder from which parent pom will be downloaded, here we need to follow a practice that 1_44_0 will have all dependencies jar version as 1.44.0-SNAPSHOT defined as the folder name suggest
2.  Secondly, build a service repository in the local repo which will not fetch dependencies from the remote using command:
    
    `mvn -Dmaven.repo.local=$HOME/.m2/local -Dversion=1_44_0 clean install`

**NOTE: Please do not include -P eshop-dev profile in the local repo as it could replace the local build dependencies jar from the corresponding Nexus repo jar.**

#### Remote Repository Development on Local Environment:
For remote building in which all dependencies will be fetched from Nexus and changes are there only in the service repository. In that case, we just need to provide Nexus profile and Parent-Pom Version. For example, if we need to build a service with current dev code using dependency jar version 1.44.0-SNAPSHOT:

    `mvn -Dmaven.repo.local=$HOME/.m2/remote -P eshop-dev -Dversion=1_44_0 clean install`

**-Dmaven.repo.local=$HOME/.m2/remote:** It tells that we need to use remote repo in the local system.

**-P eshop-dev:** It tells the profile of settings.xml which will tell the download Nexus repo for dependencies here it is dev profile.

**-Dversion:** Folder from which parent pom will be downloaded, here we need to follow a practice that 1_44_0 will have all dependency jar version as 1.44.0-SNAPSHOT defined as the folder name suggest

#### Jenkins Job Changes for Remote Environment

1. At the start of the sprint, we will publish the current sprint version in form of 1.44.0-SNAPSHOT for all jars in the parent pom
2. We will publish it by running the PARENT-POM Jenkins job which will run this command:

    `mvn deploy -Dmaven.repo.local=$HOME/.m2/local -f com.dt.pom -P parent-pom -Dversion=1_44_0`
    
    **-P parent-pom:** It tells to use distribution management defined in the parent-pom profile
    
    **-Dversion=1_44_0:** This will publish the parent-pom file in the 1_44_0 folder in the eshop-common repo as defined in parent pom above

3. Once the changes of dependencies on the dev branch are merged then the DEV Jenkins job could be triggered and that will run the command:
    
    `mvn deploy -Dmaven.repo.local=$HOME/.m2/local -P eshop-dev-dependency -DnexusPath=eshop-dev -Dversion=1_44_0`
    
    **-P eshop-dev-dependency:** It tells to use distribution management defined in jars profile
    
    **-DnexusPath=eshop-dev:** This will publish the jars to the dev folder in maven snapshots as defined in parent pom above.
    
    **-Dversion=1_44_0:** This is the parent pom folder of eshop-common of Nexus repo from which the parent-pom will be downloaded.

4. Now, to build the service repository we need to pass the profile provided in settings.xml and this can be passed as a parameter in the DEV Jenkins job :
    
    `mvn -Dmaven.repo.local=$HOME/.m2/remote -P eshop-dev -Dversion=1_44_0 clean install`
    
    This command will download the jar file from the nexus repo defined in the dev profile of settings.xml

There are three new parameters introduced in Jenkins job for this:
1. **parentpomVersion:** This is the folder name of parent pom
2. **nexusProfile:** Nexus repo from where the dependencies will be downloaded
3. **feature:** It signifies to use of local or remote dependencies

    * **Yes:** Use the dependencies that are built using dependencies job and use .m2/eshop-dev/local folder
    
    * **No:** Download the dependencies from Nexus Repo and use .m2/eshop-dev/remote folder

#### Benefits of using this approach

* In this solution, there is no commit in service repo pom to change dependency jar version or parent-pom version.
* All jar versions are maintained in a single place i.e. parent-pom
* As jar versions are denoted as SNAPSHOT they can be force replaced if the jar is updated by -U parameter
* parent-pom can be used in all environments from a single location.

#### Summarising the commands for quick lookup

1. Change version in all pom file in multi-module repository:

    `mvn versions:set -DnewVersion=1.44.0-SNAPSHOT -Dmaven.repo.local=$HOME/.m2/local -Dversion=1_44_0`

2. Dependency local build ( With exclusion of "test-web-app" module ):

    `mvn -pl '!test-web-app' -Dmaven.repo.local=$HOME/.m2/local -Dversion=1_44_0 clean install`

3. Service local build :
    
    `mvn -Dmaven.repo.local=$HOME/.m2/local -Dversion=1_44_0 clean install`

    **NOTE: Please do not include -P eshop-dev profile in the local repo as it could replace the local build dependencies jar from the corresponding Nexus repo jar.**

4. Service remote build:

    `mvn -Dmaven.repo.local=$HOME/.m2/remote -P eshop-dev -Dversion=1_44_0 clean install`


#### Intellij Changes
In Intellij to reload jar in External Dependencies

**Maven Local Repository**
1) First select local repo in Intellij

    Point maven local repository

2) Set parent-pom folder as an argument

    -Dversion = parant-pom version

3) Deselect all profiles just common must be selected

    Common must only be selected

4) Click Reimport option
    
    Click Reimport to update external library jars

**Maven Remote Repository**

1) First, change the path of the maven repo to remote

2) Set parent-pom folder as an argument

3) Select settings.xml profile like dev, the common profile is selected by default as it is the default active profile

4) Click Reimport option


The jar re-importing can be done in parallel to the building of the service repository.

This is all about the maven dependency version control for multiple services and the way we solved this in OneShop. These changes are working smoothly for the past 1 year without any issues. Indeed, It was a very interesting problem to solve. Hope this article helped in some of the learnings!
