---
layout: post
title:  "Self Contained Projects with Gradle Wrapper"
author: karl
date: 2017-06-03
comment: true
categories: [Articles]
published: true
noindex: false
---
The Gradle Wrapper is a handy way of bundling a Gradle runtime with your project.
That way you provide a specific version of Gradle to be used with your project and Gradle does not have to be installed separately.
This is very useful for anyone who clones your repo and wants to build your project.

To generate a Gradle Wrapper for your project you will need to have Gradle installed.
This is necessary to first generate the Gradle Wrapper but from then on the Wrapper can be used to run Gradle commands against your project without having to have Gradle installed.

You can find a sample project using the Gradle Wrapper [here](https://github.com/karlkyck/spring-boot-completablefuture)


### Install Gradle
Begin by installing Gradle. This is a simple as 

* Ensuring you have Java 7 or higher
* Downloading a Gradle distribution
* Unzipping the Gradle distribution to a location of your choice
* Adding the <gradle-install-path/bin folder to your PATH

Official instructions can be found [here](https://gradle.org/install)

### Generate the Gradle Wrapper

To generate a Gradle Wrapper for your project and make it self contained execute the following command in the root folder of the project:
 
```
gradle wrapper --gradle-version 3.5
```

Be sure to set the `--gradle-version` flag to the version of Gradle you wish to use with your project.

### Add Generated Files to VCS
Now that the Gradle Wrapper has been generated for you project you will see the following new files:

```
<your project folder>/
  gradlew
  gradlew.bat
  gradle/wrapper/
    gradle-wrapper.jar
    gradle-wrapper.properties
```

All of the above files must be checked into your version control system.

### Add SHA-256 Checksum Verification
For added security and to ensure the integrity of the Gradle distribution downloaded by the Gradle Wrapper you can configure the SHA-256 checksum verification.
To do this you will need to generate a SHA-256 hash of the target distribution:
 
```
> shasum -a 256 gradle-3.5-all.zip 
d84bf6b6113da081d0082bcb63bd8547824c6967fe68704d1e3a6fde822b7212  gradle-3.5-all.zip
```
  
Add the hash sum to your project's `gradle-wrapper.properties` file with the `distributionSha256Sum` property:

`distributionSha256Sum=d84bf6b6113da081d0082bcb63bd8547824c6967fe68704d1e3a6fde822b7212`

### Run the Gradle Wrapper

Run the Gradle Wrapper to make sure everything is working by executing the following command:

`./gradlew clean`

The Gradle Wrapper should download the target Gradle distribution, verify the integrity of the file using the SHA-256 checksum and then run the specified command.
