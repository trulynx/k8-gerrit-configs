# Creating and configuring an Gerrit Deployment authenticating against an LDAP database using Kubernetes

This project details steps used to set up gerrit deployment using kubernetes.


## Table of Contents
- [Creating an LDAP container](#LDAP)
- [Creating an LDAP-ADMIN container](#LDAP-ADMIN)
- [Configuring the LDAP-ADMIN container](#CONFIGURE-LDAP-ADMIN)
- [Creating a MYSQL container](#MYSQL)
- [Creating a GERRIT container](#GERRIT)

## LDAP
#### 1. create a PersistentVolumeClaim using persistent disks
    a. create a pvc manifest file
    cat > ldap-volumeclaim.yaml

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: ldap-volumeclaim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 200Gi

    b. deploy the manifest
    kubectl claim -f ldap-volumeclaim.yaml

    c. check to see if the claim has been bound
    kubectl get pvc

#### 2. create a Kubernetes secret to store the password for the admin user
    a. kubectl create secret generic ldap --from-literal=password=<your-admin-password>

#### 3. create an ldap Deployment
    a. create a manifest for the deployment
    cat > ldap.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ldap
      labels:
        app: ldap
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ldap
      template:
        metadata:
          labels:
            app: ldap
        spec:
          containers:
            - name: ldap
              image: docker.io/accenture/adop-ldap:latest
              ports:
                - containerPort: <your-port-number>
              volumeMounts:
                - name: ldap-persistent-storage
                  mountPath: /var/lib/ldap
              env:
                - name: SLAPD_DOMAIN
                  value: btech.net
                - name: SLAPD_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: ldap
                      key: password
                - name: SLAPD_FULL_DOMAIN
                  value: dc=btech,dc=net
                - name: INITIAL_ADMIN_USER
                  value: user
                - name: INITIAL_ADMIN_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: ldap
                      key: password
          volumes:
            - name: ldap-persistent-storage
              persistentVolumeClaim:
                claimName: ldap-volumeclaim
    b. deploy the manifest
    kubectl create -f ldap.yaml

    c. check its health, it might take a couple of minutes
    kubectl get pod -l app=ldap

#### 4. create a Service to expose the ldap container and make it accessible from the ldap-admin container
    a. cat > ldap-service.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: ldap
      labels:
        app: ldap
    spec:
      type: ClusterIP
      ports:
        - port: <your-port-number>
          targetPort: 389
      selector:
        app: ldap

    b. deploy the manifest and launch the service
    kubectl create -f ldap-service.yaml

    c. check the health of the created service
    kubectl get service ldap
    
#### 5. view more details about the deployment
    a. kubectl describe deploy -l app=ldap

#### 6. view more details about the pod
    a. kubectl describe po -l app=ldap

## LDAP-ADMIN
#### 1. create an ldap-admin Deployment
    a. create a manifest for the deployment
    cat > ldap-admin.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ldap-admin
      labels:
        app: ldap-admin
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ldap-admin
      template:
        metadata:
          labels:
            app: ldap-admin
        spec:
          containers:
            - name: ldap-admin
              image: docker.io/dinkel/phpldapadmin:latest
              ports:
                - containerPort: <your-port-number>
              volumeMounts:
                - name: ldap-admin-persistent-storage
                  mountPath: /var/lib/ldap-admin
              env:
		- name: LDAP_SERVER_HOST
		  value: ldap:389
		  
    b. deploy the manifest
    kubectl create -f ldap-admin.yaml

    c. check its health, it might take a couple of minutes
    kubectl get pod -l app=ldap-admin

#### 4. create a Service to expose the ldap-admin container and make it accessible from the public
    a. cat > ldap-admin-service.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: ldap-admin
      labels:
        app: ldap-admin
    spec:
      type: LoadBalancer
      ports:
        - port: <your-port-number>
          targetPort: 80
          protocol: TCP
      selector:
        app: ldap-admin

    b. deploy the manifest and launch the service
    kubectl create -f ldap-admin-service.yaml

    c. check the health of the created service
    kubectl get service ldap-admin
    
#### 5. view more details about the deployment
    a. kubectl describe deploy -l app=ldap-admin

#### 6. view more details about the pod
    a. kubectl describe po -l app=ldap-admin

#### 7. visit the app in the browser using the EXTERNAL-IP value obtained from Service details

## CONFIGURE-LDAP-ADMIN
#### 1. create a posix group (e.g. gerrit-users) so as to be able to create user accounts
    a. click "ou=groups"
    b. click "create a child entry"
    c. choose "Generic: Posix Group"
    d. click "Create Object" button to proceed
    e. on next screen click "Commit" button to persist the changes

#### 2. create users for use on gerrit
    a. click "ou=people"
    b. click "create a child entry"
    c. choose "Generic: User Account"
    d. fill in "gid", "lastname", "login shell", "password"
    e. click "Create Object" button to proceed
    f. on next screen click "Commit" to persist the changes
    g. click "Add new attribute" to add "Email" and "displayName" attributes
#### N.B. the first login on gerrit becomes the admin user so choose a "cn" wisely

