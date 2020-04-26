---
layout: single
classes: wide
title:  "Jenkins Shared Library"
description: How to create a Jenkins Shared Library.
date:   2020-03-30 12:43:00 +0200
excerpt_separator: <!--more-->
header:
  teaser: /assets/images/power_jenkins.png

tags: [jenkinsfile, jenkins, ci, cd, groovy]

---

{% include image_width.html url="/assets/images/jenkins.png" description="" width="400px" %}


Every body in the IT world know what is the CI and its famous implementation, that is **Jenkins**.<!--more--> In the last years the DevOps engineer can organize 
the continous integration *chain* as a pipeline introducing a **DSL** (domain specific language) to manage it throught the *Jenkinsfile*:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
            }
        }
    }
}
```

The following link [https://jenkins.io/doc/book/pipeline/jenkinsfile/](https://jenkins.io/doc/book/pipeline/jenkinsfile/) explains what are its syntax and features, but  in this post I want introduce you the [Jenkins Shared Library](https://jenkins.io/doc/book/pipeline/shared-libraries/) with a pratical example.

As shown in the previous script, the Jenkinsfile can become extremly unreadeable, to avoid it you can write a library to wrap your jobs. Let's start from the following pipeline:

```groovy
pipeline {
    agent any 
    tools {
        jdk 'jdk11'
    }
    
    stages {
        stage('Checkout') { 
            steps {
                ansiColor('xterm') {
                    checkout scm
                }
            }
        }
        stage('Build') { 
            steps {
                ansiColor('xterm') {
                    withMaven(maven:'maven3.6.3'){
                        sh 'mvn clean package'
                    }
                }
            }
        }
        
    }
}
```

A simple checkout and build already become unreadble, let's think what happen if you start adding other stages like quality or deployments steps. To simplify the pipeline you could use a Jenkins Shard library but let's start from beginning.
In order to execute the previous pipeline you need to install on Jenkins:

* JDK11
* Maven
* [Pipeline Maven Integration](https://plugins.jenkins.io/pipeline-maven/)

To install JDK11 I installed also [AdoptOpenJDK installer](https://plugins.jenkins.io/adoptopenjdk/), as shown in the image, now Jenkins is able to download the JDK11 from the section *Global instrument* (I set the label to *'jdk11'*)

{% include image.html url="/assets/images/jenkins_jdk11.png" description=""  %}

Then I installed Maven 3.6.3 and labelled it with *'maven3.6.3'*

{% include image.html url="/assets/images/jenkins_maven.png" description=""  %}

Now you can execute the previous pipeline...but can we write the previous pipeline in a better way like the following snippet?

```groovy
@Library('jenkins-lib@1.0.0') _

jenkinslib(this, "test")
    .withMavenVersion("maven3.6.3")
    .withJdkVersion("jdk11")
    .execute()
```

Yes we can through the Jenkins Shared library functionality! The first line indicates the library reference: to reference a library you need to install it through the Jenkins global configuration. Usally the library must be located on the SCM repository, I put my library on [this github repo](https://github.com/paspao/jenkins-lib/) then I configured it on Jenkins giving the name *'jenkins-lib'*, I also enabled the *'Discover tags'*.

{% include image.html url="/assets/images/jenkins_shdlib.png" description=""  %}

Now I can reference my library in a Jenkinsfile like **@Library('jenkins-lib@1.0.0')** or *@Library('jenkins-lib@master')* or simply *@Library('jenkins-lib')* using a specific version through a tag or a branch. After the library notation there is an underscore **_**, it means that we want to include everything into the currente pipiline (otherwise we have to put a sepcific *import*, but it isn't our case).

Let's see the library structure:

{% include image_width.html url="/assets/images/jenkinsStructure.png" description=""  width="300px" %}

I defined a directory __vars__ required by Jenkins Library structure that contains a groovy script _jenkinslib.groovy_, in this directory every .groovy file could be referenced by the _Jekinsfile_. In my example the line _jenkinslib(...)_ it is a reference to _jenkinslib.groovy_ file that contains a special function _call(...)_

```groovy
import org.paspao.sharedlib.JenkinsBuilder

JenkinsBuilder call(def scriptReference, String projectName) {
    return new JenkinsBuilder(scriptReference,projectName,env['BRANCH_NAME'])
}

return this
```
In this function I simply create an object _JenkinsBuilder_ with some initialization parameters, but the real news is that I'm moving my pipiline from a functional programming to an Object Oriented Programming giving more readibility to the pipeline code.
The magic is realized using the Closure object that (getting the Groovy definition) _is an open, anonymous, block of code that can take arguments, return a value and be assigned to a variable_.

[GitHub jenkins-lib](https://github.com/paspao/jenkins-lib/)