---
title: "Publish a Kotlin lib with gradle Kotlin DSL"
date: 2019-02-08T15:03:53+02:00
draft: false
---

I wanted to play more with Kotlin and I wanted to publish [KGeoGson](https://github.com/bastienpaulfr/geojson-kotlin) lib to a remote maven repo.

I was following [gradle guide](https://guides.gradle.org/building-kotlin-jvm-libraries/) to build my kotlin project with Kotlin DSL.

## Bootstrap Kotlin project

Create project directory with files of your library you want to build and publish. You may have a directory structure like the following:

```
project
├── build.gradle.kts
├── settings.gradle.kts
└── my-kotlin-library
    ├── build.gradle.kts
    └── src
        ├── main
        │   └── kotlin
        │       └── org
        │           └── example
        │               └── MyLibrary.kt
        └── test
            └── kotlin
                └── org
                    └── example
                        └── MyLibraryTest.kt
```

## Setup

Root **build.gradle.kts**

```kotlin
plugins {
   `build-scan`
}

buildScan {
   termsOfServiceUrl = "https://gradle.com/terms-of-service"
   termsOfServiceAgree = "yes"

   publishAlways()
}

val clean by tasks.creating(Delete::class) {
   delete(rootProject.buildDir)
}
```

In this file, « build-scan » plugin is activated and a clean task is added. That’s all for it

**settings.gradle.kts**

```kotlin
include("lib")
```

In this file, we define our modules

**lib/build.gradle.kts**

First we are adding some plugins

```kotlin
plugins {
   // Add Kotlin plugin to build our Kotlin lib
   kotlin("jvm") version "1.3.21"
   // Get version from git tags
   id("fr.coppernic.versioning") version "3.1.2"
   // Documentation for our code
   id("org.jetbrains.dokka") version "0.9.17"
   // Publication to bintray
   id("com.jfrog.bintray") version "1.8.4"
   // Maven publication
   `maven-publish`
}
```

Then we are defining dependencies and their repositories for our code

```kotlin
repositories {
   jcenter()
   mavenCentral()
}

dependencies {
   implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk7")
   implementation("com.google.code.gson:gson:2.8.5")

   testCompile("junit:junit:4.12")
}
```

We need some more tasks to add sources and javadoc to our lib. We are starting by configuring dokka task:

```kotlin
tasks {
   dokka {
       outputFormat = "html"
       outputDirectory = "$buildDir/javadoc"
       moduleName = rootProject.name
   }
}
```

We can then bundle documentation into a jar

```kotlin
val dokkaJar by tasks.creating(Jar::class) {
   group = JavaBasePlugin.DOCUMENTATION_GROUP
   description = "Assembles Kotlin docs with Dokka"
   archiveClassifier.set("javadoc")
   from(tasks.dokka)
   dependsOn(tasks.dokka)
}
```

We are creating another jar containing sources

```kotlin
val sourcesJar by tasks.creating(Jar::class) {
   archiveClassifier.set("sources")
   from(sourceSets.getByName("main").allSource)
}
```

At this stage, you are able to compile your lib. The most important part of this article begins. Let’s see how publication is working. It is very important to configure pom.xml file of maven artifact in a right manner. Otherwise you will not be able to submit your library into JCenter repo.

Let’s start configuring base of maven publish plugin

```kotlin
val artifactName = "libname"
val artifactGroup = "org.example"

publishing {
   publications {
       create<MavenPublication>("lib") {
           groupId = artifactGroup
           artifactId = artifactName
           // version is gotten from an external plugin
           version = project.versioning.info.display
           // This is the main artifact
           from(components["java"])
           // We are adding documentation artifact
           artifact(dokkaJar)
           // And sources
           artifact(sourcesJar)
       }
   }
}
```

Now we need to add information about package in **pom.xml** file. You can edit **pom.xml** with `pom.withXml {` code


```kotlin
val pomUrl = "..."
val pomScmUrl = "..."
val pomIssueUrl = "..."
val pomDesc = "..."
val pomScmConnection = ""..."
val pomScmDevConnection = "..."

val githubRepo = "..."
val githubReadme = "..."

val pomLicenseName = "The Apache Software License, Version 2.0"
val pomLicenseUrl = "http://www.apache.org/licenses/LICENSE-2.0.txt"
val pomLicenseDist = "repo"

val pomDeveloperId = "..."
val pomDeveloperName = "..."


publishing {
   publications {
       create<MavenPublication>("lib") {
           [...]

           pom.withXml {
               asNode().apply {
                   appendNode("description", pomDesc)
                   appendNode("name", rootProject.name)
                   appendNode("url", pomUrl)
                   appendNode("licenses").appendNode("license").apply {
                       appendNode("name", pomLicenseName)
                       appendNode("url", pomLicenseUrl)
                       appendNode("distribution", pomLicenseDist)
                   }
                   appendNode("developers").appendNode("developer").apply {
                       appendNode("id", pomDeveloperId)
                       appendNode("name", pomDeveloperName)
                   }
                   appendNode("scm").apply {
                       appendNode("url", pomScmUrl)
                       appendNode("connection", pomScmConnection)
                   }
               }
           }
       }
   }
}
```

Now that your maven publication is well configured, you can configure bintray plugin

```kotlin
bintray {
   // Getting bintray user and key from properties file or command line
   user = if (project.hasProperty("bintray_user")) project.property("bintray_user") as String else ""
   key = if (project.hasProperty("bintray_key")) project.property("bintray_key") as String else ""

   // Automatic publication enabled
   publish = true

   // Set maven publication onto bintray plugin
   setPublications("lib")

   // Configure package
   pkg.apply {
       repo = "maven"
       name = rootProject.name
       setLicenses("Apache-2.0")
       setLabels("Gson", "json", "GeoJson", "GPS", "Kotlin")
       vcsUrl = pomScmUrl
       websiteUrl = pomUrl
       issueTrackerUrl = pomIssueUrl
       githubRepo = githubRepo
       githubReleaseNotesFile = githubReadme

       // Configure version
       version.apply {
           name = project.versioning.info.display
           desc = pomDesc
           released = Date().toString()
           vcsTag = project.versioning.info.tag
       }
   }
}
```

Here we is ! Happy publication !
