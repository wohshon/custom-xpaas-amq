## Customizing xPaaS AMQ 6

#### 1. Overview

In this demo, we are using s2i to apply customizations 
to AMQ 6 xPaaS Images.

The following customizations will be applied
1. create destinations on startup
  - all mqtt topics
2. RBAC for the topics created previously
3. Loading a customized plugin to intercept the addConnection and addConsumer usecases


#### 2. Configuring activemq.xml and other properties

- Changes to `activemq.xml` is to be loaded into a file call `openshift-activemq.xml` and
placed into a folder call `configuration`

- Additional jar files to be placed in a `lib` folder

- Destinations to be created during startup are specified in the 
`<destinations>` section.

    <destinations>
        <topic physicalName="demo.finance.job" />
        <topic physicalName="demo.it.request" />
        <topic physicalName="demo.hr.newhire" />
    </destinations>


#### 3. Configuring authentication and plugins

We are using the default `PropertyLoginModule` in the image.
It means a  `login.config` is present in `/opt/amq/conf` which
points to `users.properties` and `groups.properties`

Changes to `users.properties` is to be pushed in via` openshift-users.properties`
This file will be placed `configuration`

    #
    # You must have at least one users to be able to access JBoss A-MQ resources

    ##### AUTHENTICATION #####
    admin=admin
    hr-user=password
    finance-user=password
    it-user=password
    ops-user=password
    superUser=password

Changes to `groups.properties` can be pushed in via the same file
This file will be placed `configuration`

    #
    # This file contains the roles for users who can log into JBoss A-MQ.
    # Each line has to be of the format:
    #
    # GROUP=USER1,USER2,....
    #
    admin=admin,superUser
    hr=hr-user
    finance=finance-user
    it=it-user
    operation=ops-user
    users=admin,superUser,it-user,hr-user,finance-user,ops-user


Authorization to the destinations will be handled by the `<authorizationPlugin>`

    <plugins>
        <!-- ##### AUTHENTICATION ##### -->
    <bean id="customAuthenticationPlugin" class="com.demo.amqplugins.CustomAuthenticationPlugin" xmlns="http://www.springframework.org/schema/beans">
      </bean>
        <authorizationPlugin>
            <map>
                <authorizationMap>
                    <authorizationEntries>
                        <authorizationEntry topic=">" write="admin" read="admin" admin="admin" />
                        <authorizationEntry topic="demo.finance.>" write="finance" read="operation,finance" admin="admin" />
                        <authorizationEntry topic="demo.hr.>" write="hr" read="operation,hr" admin="admin" />
                        <authorizationEntry topic="demo.it.>" write="it" read="operation,it" admin="admin" />
                        <authorizationEntry topic="ActiveMQ.Advisory.>" read="users" write="users" admin="users"/>
                    </authorizationEntries>
                </authorizationMap>
            </map>
        </authorizationPlugin>
    </plugins>



#### 4. Deploying to openshift

#### Create a new project, e.g. `custom-amq`

    $ oc new-project custom-amq

#### Create a new image stream using the s2i build, pointing to this repo

    $ oc new-build registry.access.redhat.com/jboss-amq-6/amq63-openshift~https://github.com/wohshon/custom-xpaas-amq

#### After the build is done, a new image stream will be created.

```
$ oc get is

NAME               DOCKER REPO                                   TAGS      UPDATED
amq63-openshift    172.30.1.1:5000/custom-amq/amq63-openshift    latest    41 minutes ago
custom-xpaas-amq   172.30.1.1:5000/custom-amq/custom-xpaas-amq   latest    41 minutes ago

```    

#### Create our own template based on the default template, 2 things needs to be changed

- the image stream must point to our newly created one
- Optionally, point to a dedicated service account

  ```
  $ sed -i 's/"image": "jboss-amq-63"/"image": "172.30.1.1:5000\/custom-amq\/custom-xpaas-amq"/g' amq63-basic.json

  $ sed -i '/"terminationGracePeriodSeconds": 60,/a\                        "serviceAccountName": "amq-service-account",' amq63-basic.json

  ```

  `$ oc create -f ./amq63-basic.json`
  
  
#### Deploy new template, update deploymentConfig with the correct imageStreamTag, and deploy

```
  $ oc new-app --template="custom-amq/amq63-basic" -p MQ_USERNAME=admin -p MQ_PASSWORD=admin -p AMQ_STORAGE_USAGE_LIMIT=2gb -p  IMAGE_STREAM_NAMESPACE=custom-amq -p MQ_PROTOCOL="openwire,mqtt,amqp"

  $ oc patch dc/broker-amq --patch '{"spec":{"triggers":[{"type": "ImageChange","imageChangeParams":{"containerNames": ["broker-amq"],"from": {"name": "custom-xpaas-amq:latest"}}}]}}'

  $ oc rollout latest broker-amq
```

#### Create a nodePort to point to the MQTT port (or any protocol you desired) 

sample nodeport config

    apiVersion: v1
    kind: Service
    metadata:
      name: amq-mqtt-nodeport
      namespace: amq
      labels:
        application: broker
    spec:
      ports:
        - name: port-1
          protocol: TCP
          port: 1883
          targetPort: 1883
          nodePort: 30001 
      selector:
        application: broker
      type: NodePort 
      sessionAffinity: None

#### Testing

The mappings of the users to the topics are

- it-user : demo/it/request
- finance-user: demo/finance/job
- hr-user : demo/hr/newhire
- admin has all rights, and only they can create queue/topics

CustomPlugin:

Custom plugin intercepts and print out subscriber info

    INFO | TOPIC topic://demo.hr.newhire
    INFO | User  hr-user
    INFO | Context  org.apache.activemq.security.JaasAuthenticationBroker$JaasSecurityContext@1e129afe
    INFO | Principals  [hr, users, hr-user]
    INFO | Write Destinations  {topic://demo.hr.newhire=topic://demo.hr.newhire}
    INFO | ROLES: hr
    INFO | ROLES: users

