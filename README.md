# Conjur-OSS-Setup
This Conjur plugin securely provides credentials that are stored in Conjur to Jenkins jobs.
Step by step guide
Set up a Conjur Open Source environment
  you can refer https://www.conjur.org/get-started/quick-start/oss-environment/
Update the docker-compose file with required images and configuration by  including jenkins,cloudbees
 Add Jenkins
   jenkins_server:
    image: jenkins/jenkins:2.401.3
    container_name: jenkins_server
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 8088:8080
    volumes:
      - ./jenkins_custom_volume:/var/jenkins_home
    environment:
      - JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
      - PATH=/usr/lib/jvm/java-17-openjdk-amd64/bin:$PATH
    restart: on-failure
  Add CloudBees
       cloudbees:
        image: cloudbees/cloudbees-core-mm:2.426.2.2
        container_name: cloudbees
        ports:
          - 8085:8080
        restart: on-failure
        volumes:
           - ./cloudbees:/var/jenkins_home
Configuration using api-key
    Define the policy by including hosts and its secrets for api authentication 
      Define a workload (host) policy in Conjur to represent your workload.
         Name this policy file as jenkins-host-api.yml
```
 - !policy 
    - !host
       id: jenkins-app
    - &variables
      - !variable dbUserName
      - !variable dbPassword
      - !variable dbUrl
```
Note: Make sure you have same host as jenkins job 
    Load the policy files 

```
$ conjur policy load -f jenkins-host-api.yml -b root
```

Once you loaded the policy it will generate api-key of that host this api-key is used for authentication of host at jenkins 
Set the value of secret variables

        ```    
          $ conjur variable set -i jenkins-app/dbUserName -v "JENKINS"  
        
 ``` 
       $ conjur variable set -i jenkins-app/dbPassword -v "Password" 
 ```

    
    ``` 
       $ conjur variable set -i jenkins-app/dbUrl -v "http://google.com" 
    ```
     

Configuration using jwt authentication
    Define policy for jwt authentication
      Create a JWT authentication policy file and save it is authn-jwt-jenkins.yml
  ```
- !policy
   id: conjur/authn-jwt/jenkins
   annotations:
    description: JWT Authenticator web service for Jenkins
    jenkins: true
   body:
    # Create the conjur/authn-jwt/jenkins web service
    - !webservice

    # Optional: Uncomment any or all of the following variables:
    # * token-app-propery
    # * identity-path
    # * issuer
    # identity-path is always used together with token-app-property
    # however, token-app-property can be used without identity-path

    - !variable
      id: token-app-property
      annotations:
        description: JWT Authenticator bases authentication on claims from the JWT. You can base authentication on identifying clams such as the name, the user, and so on. If you can customize the JWT, you can create a custom claim and base authentication on this claim.

    - !variable
      id: identity-path
      annotations:
        description: JWT Authenticator bases authentication on a combination of the claim in the token-app-property and the full path of the application identity (host) in Conjur. This variable is optional, and is used in conjunction with token-app-property. It has no purpose when standing alone.

    - !variable
      id: issuer
      annotations:
        description: JWT Authenticator bases authentication on the JWT issuer. This variable is optional, and is relevant only if there is an iss claim in the JWT. The issuer variable and iss claim values must match.
    
    - !variable
      id: audience
      annotations:
        description: JWT Authenticator validates the audience (aud) in the JWT.

    # Mandatory: The JWT Provider URI: You must provide either a provider-uri or jwks-uri

    # - !variable
    #   id: provider-uri
    #   annotations:
    #     description: The JWT provider URI. Relevant only for JWT providers that support the Open ID Connect (OIDC) protocol.

    - !variable
      id: jwks-uri
     # id: public-keys
      annotations:
        description: A JSON Web Key Set (JWKS) URI. If the JWKS vendor provides both a jwks-uri and an equivalent provider-uri, you can use the provider-uri which has an easier interface to work with.

    # Group of hosts that can authenticate using this JWT Authenticator
    - !group
      id: consumers
      annotations:
        description: Allows authentication through authn-jwt/jenkins web service.
        editable: "true"
    
    # Permit the consumers group to authenticate to the authn-jwt/jenkins web service
    - !permit
      role: !group consumers
      privilege: [ read, authenticate ]
      resource: !webservice

    # Create a web service for checking authn-jwt/jenkins status
    - !webservice
      id: status

    # Group of users who can check the status of authn-jwt/jenkins
    - !group
      id: operators
      annotations:
        description: Group of users that can check the status of the authn-jwt/jenkins authenticator.
        editable: "true"
    
    # Permit group to check the status of authn-jwt/jenkins
    - !permit
      role: !group operators
      privilege: read
      resource: !webservice status
```

    
Load the policyfile of jwt authentication
    $conjur policy load  -b root -f  policy/authn-jwt-jenkins.yml
    set the variables values of authn-jwt policy file
       In case of jenkins 
         docker compose exec client conjur variable set -i conjur/authn-jwt/jenkins/jwks-uri -v 'http://jenkins_server:8080/jwtauth/conjur-jwk-set'
         docker compose exec client conjur variable set -i conjur/authn-jwt/jenkins/issuer -v 'http://localhost:8083'
       In case of cloudbees
         docker compose exec client conjur variable set -i conjur/authn-jwt/jenkins/jwks-uri -v 'http://cloudbees:8080/jwtauth/conjur-jwk-set'
         docker compose exec client conjur variable set -i conjur/authn-jwt/jenkins/issuer -v 'http://localhost:8091'
      docker compose exec client conjur variable set -i conjur/authn-jwt/jenkins/audience  -v 'cyberark-conjur'
      docker compose exec client conjur variable set -i conjur/authn-jwt/jenkins/token-app-property -v 'jenkins_name'
      docker compose exec client conjur variable set -i conjur/authn-jwt/jenkins/identity-path -v 'jenkins/projects'
    Define the hosts and also secrets and set the values to those secret variables
      1. Define hosts in policy name it as jenkins-projects.yml
         - !policy
           id: jenkins
           body:
              - !policy
                id: projects
                annotations:
                   description: Projects that do not fall under a folder within Jenkins or project-specific host identities for authn-jwt/jenkins authentication.
                   jenkins: true
                body:
                    - !host
                      id: pipeline-job
                      annotations:
                          jenkins: true
                          authn-jwt/jenkins/jenkins_pronoun: Pipeline
                          authn-jwt/jenkins/jenkins_full_name: pipeline-job
                    - !grant
                       role: !group
                       members:
                          - !host pipeline-job
          - !grant
             role: !group conjur/authn-jwt/jenkins/consumers
             member: !group jenkins/projects
                  
      2. load the policy file of hosts defined
         $ docker compose exec client conjur policy load -b root -f policy/jenkins-projects.yml
      3. Define the secrets and name it as  jenkins-secrets.yml
         - &simple-pipeline-secrets
             - !variable pipeline-credential-1
             - !variable pipeline-credential-2

        - !permit
          resources: *simple-pipeline-secrets
          privileges: [read, execute]
          roles: !host jenkins/projects/pipeline-job
      4. Set the value to those secrets
          
         $docker compose exec client conjur variable set -i pipeline-credential-1  -v "This-has-secret-value"
         $docker compose exec client conjur variable set -i pipeline-credential-2  -v "This-has-secret-value-2"

Global Configuration in jenkins
  Global Configuration: Conjur Appliance 
    1. A global configuration allows any job to use the configuration, unless a folder-level configuration overrides the global configuration. Click the Global Credentials tab. Define the Conjur Account and Appliance URL to use.
     1.1 By providing conjur appliance url and conjur Account.
     1.2 In case of API Authentication not in case of JWT authentication
           i) Provide API auth credential 
           ii) Provide SSH Certificate
      
  Global Configuration: Conjur JWT Authentication
      
      
   In case of API authentication 
     Folder/Job Property Configuration
     Conjur Login Credential (In case of not using JWT Authentication)
     Conjur Secret Definition (Static)
Usage from a Jenkins pipeline script
