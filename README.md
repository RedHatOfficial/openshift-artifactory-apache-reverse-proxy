# Openshift Artifactory Apache Reverse Proxy
This repository contains instructions and resources for creating an Apache Reverse Proxy on OpenShift for Artifactroy also running on OpenShift.
This is based on the steps outlined in https://www.jfrog.com/confluence/display/RTF/Configuring+a+Reverse+Proxy only does as much of the work for you as possible.

## Note about NGINX
Artifactory leands towards using NGIX rather then Apache for the reverse proxy, but after many days of troubleshooting we could not get NGINX reverse proxy working while sitting behind the OpenShift routers (HAProxy). Therefor this Apache Reverse Proxy aproach.

## Instrucitons

Instructions for setting up the Artifactory Apache Reverse Proxy.

### Prerequists

1. Deploy Artifactroy to OpenShift
   * Some helpful information for doing that can be found here, https://github.com/RHsyseng/artifactory-on-openshift/

### Generate Apache Reverse Proxy Configuration
Artifactory will generate a pretty good configuration base for Apache Reverse Proxy, here are the steps to do so.

1. Log into Artifactory with admin privlages
2. Admin -> Configuration -> HTTP Settings
3. Docker Settings
   * Docker Access Method: Sub Domain
   * Server Name Expression: __NOTE__: this value is not editable, but it is the one you need to generate a wildcard certificate for
4. Reverse Proxy Settings
   * Server Provider: `Apache`
   * Internal Hostname: `artifactory.artifactory.svc`
     * assuming Artifactory is deployed in the `artifactory` namespace/project
   * Internal Port: `8081`
     * assuming defualt Artifactory deployment
   * Internal Context Path: `artifactory`
     * assuming defualt Artifactory deployment
   * Public Server Name: this should be the public route to Artifactory, ex: `artifactory.apps.example.org`
   * Public COntext Path: `artifactory`
     * assuming defualt Artifactory deployment
   * Use HTTP: `NO` (not check)
   * Use HTTPS: `YES` (check)
   * HTTPS Port: `8443`
   * SSL Key Path: `/etc/ssl/tls.key`
     * the key will be mounted as a secret to this locaiton, so don't change it
   * SSL Certificate Path: `/etc/ssl/tls.crt`
     * the certificate will be mounted as a secret to this locaiton, so don't change it
5. Save
6. Download, save as `artifactory-proxy.conf` 
7. Optional: put in better logging and timeoute
   ```
cat <<'EOF' | patch artifactory-proxy.conf
@@ -16,9 +16,15 @@
     SSLProxyEngine on

     ## Application specific logs
-    ## ErrorLog ${APACHE_LOG_DIR}/artifactory.apps.mgt.devsecops.gov-error.log
-    ## CustomLog ${APACHE_LOG_DIR}/artifactory.apps.mgt.devsecops.gov-access.log combined
-
+    ErrorLog   /dev/stdout
+    CustomLog  /dev/stdout combined
+
+    ## additional logging
+    LogLevel Info
+
+    ##Timeout
+    TimeOut 300
+
     AllowEncodedSlashes On
     RewriteEngine on
EOF
   ```

###  Source Control the Apache Reverse Proxy Configuration

The S2I build of the Apache Reverse Proxy requires that the generated Apache configuration be in a Git project.

1. create a Git project on your SCM server, ex: `artifactory-apache-reverse-proxy`
2. clone the new repo
3. create a `httpd-cfg` directroy in the new repo
4. put the `artifactory-proxy.conf` file in the `httpd-cfg` repo
5. add, commit, and push the file to the repo
6. Optional: tag the repo

### Generate the Wildcard Certificate for use by the Reverse Proxy
The Apache Reverse Proxy will be servering multiple endpoints all based on the __Server Name Experssion__ from the __HTTP Settings__ page. Therefor a wildcard certificate to match that expression will be needed. Be sure the signing request and public/private key include the wildcard FQDN in both the primary and SAN.

### Deploy the Artifactory Apache Reverse Proxy

1. clone this repository
2. `cd openshift-artifactory-apache-reverse-proxy`
3. `oc project artifactroy`
   * assuming `artifactroy` is the namespace name for where artifactroy is deployed
4. `oc create -f artifactory-apache-reverse-proxy-template.yaml`
5. Log into OpenSHift
6. Go to the `artifactory` project
7. Add to Project -> Select from Project
8. Selection
   1. Artifactory Apache Reverse Proxy
     * if this option is not showing up, something went wrong with step 4
   2. Next >
9. Information
   1. Read the info
   2. Next >
10. Configuration
    1. Git Repository URL: the URL created in the __Source Control the Apache Reverse Proxy Configuration__ section
    2. Git Reference: change this from master if you created a tag, recomended
    3. Git Context Directory: if you followed these instrucitons exactly, you can leave this blank, but if you put the `httpd-cfg/artifactory-proxy.conf` somewhere other then the root of the project you will need to change this.
    4. TLS Certificate: public certificate generated in __Generate the Wildcard Certificate for use by the Reverse Proxy__
    5. TLS Key: private certificate generated in __Generate the Wildcard Certificate for use by the Reverse Proxy__
    6. Create

### Configure Routes per Artifactory Service
For each service you want to route through the reverse proxy you will need to create an OpenShift route for that service. For example one for the docker registery, another for the NPM regisetery, etc.

You would follow these steps for each service to expose, or for instance each docker registery to expose.

1. OpenShift -> My Projects -> artifactory
2. Applications -> Routes
3. Create Route
   * Name: ex: `artifactory-docker`
   * Hostname: ex: `docker.artifactory.apps.example.org`
     * this value should be based on the exposed service from Artifactory and the __Server Name Expresion__
   * Path: `/`
   * Service: `artifactory-apache-reverse-proxy`
   * Target Port: `8443 -> 8443 (TCP)`
   * Secure Route: `YES` (check)
   * TLS Termination: `Passthrough`
   * Insecure Traffic: `None` or `Redirect` (your choice)
4. Save
