# keycloak-kubernetes
secure Kubernetes Web applications with Keycloak

## Prerequisites

Make sure you have Minikube installed, ideally with the Ingress addon enabled.
Make sure you the Ingress addon enabled run:
  ```sh
  minikube addons list
  ```
If the Ingress addon is not enabled run :
 ```sh
  minikube addons enable ingress
  ```
## Setup Keycloak

### Run Keycloak
 ```sh
  kubectl apply –f keycloak.yaml
   ```
This will start 2 containers on Kubernetes.
Keycloak container runs the keyclock service with an initial admin user with username admin and password admin.
Gatekeeper container is a transparent authentication proxy that integrates with the keycloak authentication service. 

### Access Keycloak with Ingress addon enabled
Start by creating an Ingress for Keycloak:
   ```sh
   kubectl apply –f keycloack-ingress.yaml 
   ```
Run the following to find out the URLs of Keycloak:
   ```sh
   KEYCLOAK_URL=https://keycloak.$(minikube ip).nip.io/auth && echo "" &&
   echo "Keycloak:                 $KEYCLOAK_URL" &&
   echo "Keycloak Admin Console:   $KEYCLOAK_URL/admin" &&
   echo "Keycloak Account Console: $KEYCLOAK_URL/realms/myrealm/account" &&
   echo ""

   ```
### Login to the admin console

Go to the Keycloak Admin Console and login with the username and password you created earlier.

1. Create a realm
   Enter a realm name, we will use "local" then click Create. 
2. Configure an OpenID-Connect Client 
   	* Click Clients in the Sidebar and then click the Create button.
    * Enter the Client ID. We will use “gatekeeper”.
    * Select the Client Protocol “openid-connect” from the drop-down menu and click Save. You will be taken to the configuration Settings page of the “gatekeeper” client.
    * From the Access Type drop-down menu, select confidential. This is the access type for server-side applications.   
    * In the Valid Redirect URIs box, you can add multiple URLs that are valid to be redirected after the authentication.
    * Next create a mapping that adds to the generated token the “Groups” and “Audience” fields. "Audience" is required by Gatekeeper to be able to authenticate the users. The         “Groups” field is optional but it allows you to filter the group of users that have access to your application.
      Go to the “Mappers” tab and click “Create”. Select “Audience” on Mapper Type, name itand in the Included Client Audience, select the created “gatekeeper”client.
       Next, create the groups field mapping in a similar way. Click “Create”, select “Group Membership” on Mapper Type, name it and in the Token Claim Name, use “groups”. Turn off Full group path.
    * Finally, go to the “Credentials” tab to get the Secret. This is needed to configure the gatekeeper proxy sidecar container that will be configured on your application.
3. Add users and groups 
   Let's create two test users, one that is a member of a group that will have access to your application and one that is not a member of this group.
  	* Click Users in the Manage sidebar to view the user information for the current realm (Local).
  	* Click Add User.
  	* Enter a valid Username and any additional information (optional) and click Save.
  	* 	Click the Credentials tab for this user and enter a password. Ensure the Temporary option is set to Off so that it does not prompt for a password change later on, and click Set Password. 
  	* 	Now create a group. Click Groups in the sidebar then click New. Name it as you want  and click Save.
  	* 	Now create another user and click the Groups tab in the user page. On the right side, select the group and click Join.

## Deploying your application on Kubernetes
 Now we will deploy a simple NGINX web server to demonstrate a front-end Web application that will be protected behind Keycloak.
 1.	Deploy application 
    Here is a simple yaml file composed of a Deployment, a Service and an Ingress. 
   ```sh
   kubectl apply –f keycloack-ingress.yaml 
   ```
   
  before testing , you must update the keycloak .yaml file specialy Gatekeeper configMaps 
   * discovery-url is the URL of your Keycloak server with /auth/realms/[realm_name] at the end. I used the nip.io service here too.
   * skip-openid-provider-tls-verify since Keycloak has no valid certificates, we have this as true
   * client-id is the client ID we obtained when creating the "gatekeeper" client on Keycloak
   * client-secret is the Secret we obtained when creating the "gatekeeper" client on Keycloak
   * redirection-url is the URL used by this application. The same configured in the Ingress resource above.
   * secure-cookie is set to false since our exposed application (in redirection-url) is HTTP instead of HTTPS.
   * upstream-url is the URL gatekeeper will forward the traffic to.
   * And the resources part is an optional one where I tell gatekeeper to only allow users to members of the group "my-app" the access any page in this application. 
2.	Testing 
   ```sh
   Minikube service ngnix -- url 
   ```
And the keycloak interface will appear , you must login with the authorized  user and you will find your service nginx 


