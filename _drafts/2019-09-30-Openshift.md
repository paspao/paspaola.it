---
layout: single
classes: wide
title:  "Build and deploy with Openshift"
description: A simple pipeline with Openshift.
date:   2019-09-16 16:44:00 +0200
excerpt_separator: <!--more-->
header:
  teaser: /assets/images/openshift.png

tags: [docker, openshift, jenkins, ci, cd]

---

{% include image_width.html url="/assets/images/openshift.png" description="" width="600px" %}



```
pipeline { 
    agent {
      label 'maven'
    }
    stages { 
	stage ('Initialize') {
            steps {
                sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                '''
            }
    }
    stage('Build') { 
	    steps{
               echo 'Stato Remi Service build' 
	           sh 'mvn clean package  -DskipTests'
            }
        }
    

    stage('Build Image') { 
        steps{
            echo 'Stato Remi Service Image build' 
            script{
            openshift.withCluster("ocp-noprod") {
                openshift.withCredentials("jenkins-ci-cd-noprod") {
                         openshift.withProject('1499-d-summer') {
                            def bld = openshift.startBuild('summer-stato-remi-service-build','--from-file=target/stato-remi-service-1.0-SNAPSHOT.jar')
                            bld.untilEach {
                              return it.object().status.phase == "Running"
                            }
                            bld.logs('-f')
                         }  
                     }
                }
            }
            }
  }
    }
	post {
        
        failure
        {
            mail to: 'Pasquale.Paola@eng.it,Claudio.Bevilacqua@eng.it,Massimiliano.Picca@eng.it,Ciro.DAlessandro@eng.it,Francesco.Cuomo@eng.it,Mirko.Flaminio@eng.it',
            from: 'jenkins@snam.com',
            subject: "Status of pipeline:  ${env.JOB_NAME} ${currentBuild.fullDisplayName}",
            body: "${env.BUILD_URL}  has result ${currentBuild.result}"
        }

        unstable {
            mail to: 'Pasquale.Paola@eng.it,Claudio.Bevilacqua@eng.it,Massimiliano.Picca@eng.it,Ciro.DAlessandro@eng.it,Francesco.Cuomo@eng.it,Mirko.Flaminio@eng.it',
            from: 'jenkins@snam.com',
            subject: "Status of pipeline:  ${env.JOB_NAME} ${currentBuild.fullDisplayName}",
            body: "${env.BUILD_URL}  has result ${currentBuild.result}"
         }
         changed {
           mail to: 'Pasquale.Paola@eng.it,Claudio.Bevilacqua@eng.it,Massimiliano.Picca@eng.it,Ciro.DAlessandro@eng.it,Francesco.Cuomo@eng.it,Mirko.Flaminio@eng.it',
            from: 'jenkins@snam.com',
            subject: "Status of pipeline:  ${env.JOB_NAME} ${currentBuild.fullDisplayName}",
            body: "${env.BUILD_URL}  has result ${currentBuild.result}"
         }
    }	
    }

```

```bash
oc create imagestream $nomeImmagine:tag
```

```yml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  creationTimestamp: '2019-07-30T16:19:50Z'
  name: summer-stato-remi-service-build
  namespace: 1499-d-summer
  resourceVersion: '149616810'
  selfLink: >-
    /apis/build.openshift.io/v1/namespaces/1499-d-summer/buildconfigs/summer-stato-remi-service-build
  uid: db543764-b2e5-11e9-85a1-005056ba4881
spec:
  failedBuildsHistoryLimit: 5
  nodeSelector: null
  output:
    to:
      kind: ImageStreamTag
      name: 'stato-remi-service:latest'
  postCommit: {}
  resources:
    requests:
      cpu: 50m
      memory: 100Mi
  runPolicy: Serial
  source:
    dockerfile: >-
      FROM redhat-openjdk-18/openjdk18-openshift

      LABEL maintainer="Pasquale Paola <pasquale.paola@eng.it>"

      COPY *.jar app.jar

      EXPOSE 8090

      CMD ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar",
      "app.jar","--spring.datasource.url=jdbc:sqlserver://${DATABASE_SERVER};database=summer_stato_remi","--spring.datasource.username=${DB_USER}","--spring.datasource.password=${DB_PASSWORD}","--spring.kafka.bootstrap-servers=${KAFKA_SERVERS}","--server.port=${SERVER_PORT}",
      "--jwt.secret=${JWT_SECRET}", "--jwt.profiler-url=${JWT_PROFILER_URL}",
      "--jwt.enabled=${JWT_ENABLED}"]
    type: Dockerfile
  strategy:
    dockerStrategy:
      noCache: true
    type: Docker
  successfulBuildsHistoryLimit: 5
  triggers: []
status:
  lastVersion: 42

```





```yml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftWebConsole
  creationTimestamp: '2019-07-30T16:21:03Z'
  generation: 46
  labels:
    app: stato-remi-service
  name: stato-remi-service
  namespace: 1499-d-summer
  resourceVersion: '149617207'
  selfLink: >-
    /apis/apps.openshift.io/v1/namespaces/1499-d-summer/deploymentconfigs/stato-remi-service
  uid: 06b155ab-b2e6-11e9-85a1-005056ba4881
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    app: stato-remi-service
    deploymentconfig: stato-remi-service
  strategy:
    activeDeadlineSeconds: 21600
    resources:
      limits:
        cpu: '2'
        memory: 1Gi
      requests:
        cpu: 20m
        memory: 50Mi
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      timeoutSeconds: 1200
      updatePeriodSeconds: 1
    type: Rolling
  template:
    metadata:
      annotations:
        openshift.io/generated-by: OpenShiftWebConsole
      creationTimestamp: null
      labels:
        app: stato-remi-service
        deploymentconfig: stato-remi-service
    spec:
      containers:
        - env:
            - name: DATABASE_SERVER
              valueFrom:
                configMapKeyRef:
                  key: db.url
                  name: database-summer-svil
            - name: DB_USER
              valueFrom:
                configMapKeyRef:
                  key: db.username
                  name: database-summer-svil
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: db-svil-pssword
            - name: KAFKA_SERVERS
              valueFrom:
                configMapKeyRef:
                  key: kafka-servers
                  name: kafka-svil
            - name: SERVER_PORT
              valueFrom:
                configMapKeyRef:
                  key: server.port
                  name: serverport-summer-svil
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  key: secret
                  name: jwt-svil
            - name: JWT_PROFILER_URL
              valueFrom:
                configMapKeyRef:
                  key: profiler-url
                  name: jwt-svil
            - name: JWT_ENABLED
              valueFrom:
                configMapKeyRef:
                  key: enabled
                  name: jwt-svil
          image: >-
            docker-registry.default.svc:5000/1499-d-summer/stato-remi-service@sha256:983d47dfb9330a6da2cde5a378fdbdd7ca46cebff73d8579a54d1c4d0ecd8b8b
          imagePullPolicy: Always
          name: stato-remi-service
          ports:
            - containerPort: 8090
              protocol: TCP
          resources:
            limits:
              cpu: '2'
              memory: 4Gi
            requests:
              cpu: 50m
              memory: 100Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
  test: false
  triggers:
    - imageChangeParams:
        automatic: true
        containerNames:
          - stato-remi-service
        from:
          kind: ImageStreamTag
          name: 'stato-remi-service:latest'
          namespace: 1499-d-summer
        lastTriggeredImage: >-
          docker-registry.default.svc:5000/1499-d-summer/stato-remi-service@sha256:983d47dfb9330a6da2cde5a378fdbdd7ca46cebff73d8579a54d1c4d0ecd8b8b
      type: ImageChange
    - type: ConfigChange
status:
  availableReplicas: 1
  conditions:
    - lastTransitionTime: '2019-09-16T16:17:24Z'
      lastUpdateTime: '2019-09-16T16:17:24Z'
      message: Deployment config has minimum availability.
      status: 'True'
      type: Available
    - lastTransitionTime: '2019-09-17T12:57:46Z'
      lastUpdateTime: '2019-09-17T12:57:48Z'
      message: replication controller "stato-remi-service-39" successfully rolled out
      reason: NewReplicationControllerAvailable
      status: 'True'
      type: Progressing
  details:
    causes:
      - imageTrigger:
          from:
            kind: DockerImage
            name: >-
              docker-registry.default.svc:5000/1499-d-summer/stato-remi-service@sha256:983d47dfb9330a6da2cde5a378fdbdd7ca46cebff73d8579a54d1c4d0ecd8b8b
        type: ImageChange
    message: image change
  latestVersion: 39
  observedGeneration: 46
  readyReplicas: 1
  replicas: 1
  unavailableReplicas: 0
  updatedReplicas: 1

```






```yml
apiVersion: v1
kind: Service
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftWebConsole
  creationTimestamp: '2019-07-30T16:21:46Z'
  labels:
    app: stato-remi-service
  name: stato-remi-service
  namespace: 1499-d-summer
  resourceVersion: '124473299'
  selfLink: /api/v1/namespaces/1499-d-summer/services/stato-remi-service
  uid: 20176022-b2e6-11e9-85a1-005056ba4881
spec:
  clusterIP: 172.30.211.71
  ports:
    - name: 8090-tcp
      port: 8090
      protocol: TCP
      targetPort: 8090
  selector:
    deploymentconfig: stato-remi-service
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}

```



```yml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  annotations:
    openshift.io/host.generated: 'true'
  creationTimestamp: '2019-08-09T10:54:30Z'
  labels:
    app: stato-remi-service
  name: api-route-stato-remi-service
  namespace: 1499-d-summer
  resourceVersion: '129139152'
  selfLink: >-
    /apis/route.openshift.io/v1/namespaces/1499-d-summer/routes/api-route-stato-remi-service
  uid: 1087ad8c-ba94-11e9-9f56-005056ba64fc
spec:
  host: api-route-stato-remi-service-1499-d-summer.apps.ocp-gc.snamretegas.priv
  port:
    targetPort: 8090-tcp
  to:
    kind: Service
    name: stato-remi-service
    weight: 100
  wildcardPolicy: None
status:
  ingress:
    - conditions:
        - lastTransitionTime: '2019-08-09T10:54:30Z'
          status: 'True'
          type: Admitted
      host: api-route-stato-remi-service-1499-d-summer.apps.ocp-gc.snamretegas.priv
      routerName: router
      wildcardPolicy: None

```
