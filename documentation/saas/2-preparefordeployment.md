# Prepare for Deployment in the SAP BTP, Cloud Foundry Runtime

## Update the MTA File with Multitenancy Configuration 

To deploy a multitenant application and access it in the subscriber subaccount through SAP Build Work Zone, there is a need to update the MTA configuration for design time and runtime configurations. 
In the `mta.yaml` file, update the following configurations:

1. Add dependencies to `incidents-mtx`. To get the reuse dependent services, add the following services to the required section:
  
  ```yaml
    - name: incidents-mtx
        type: nodejs
        path: gen/mtx/sidecar
        build-parameters:
        builder: npm-ci
        parameters:
        memory: 256M
        disk-quota: 512M
        provides:
        - name: mtx-api
            properties:
            mtx-url: ${default-url}
        requires:
        - name: incidents-db
        - name: incidents-auth
        - name: incidents-registry
        - name: incident-management-repo-host # Add
  ```
  
> **NOTE** 
> Steps 2-6 are optional and are not required if you have already deployed Single Tenant Application with Workzone using CDM 

2. Add `html5-repo-runtime` under `resources`:

   ```yaml
      resources:
      ...
      - name: incidents_html_repo_runtime
        type: org.cloudfoundry.managed-service
        parameters:
           service: html5-apps-repo
           service-name: incidents-html5-app-runtime-service
           service-plan: app-runtime
   ```
   
3. Update  `incident-management-destination-content`:
   
   1. Add
        ```yaml
            - name: incidents_html_repo_runtime
                parameters:
                service-key:
                    name: incidents-html5-app-runtime-service-key
        ```
   
   2. Update  `parameters->content->instance` to `parameters->content->subaccount`.
   3. Delete the destinations under it: 
        
        ```yaml
        - Name: ns_incidents_wz_capire_incidents_repo_host
          ServiceInstanceName: incident-management-html5-srv
          ServiceKeyName: incident-management-repo-host-key
          sap.cloud.service: ns.incidents.wz
        - Authentication: OAuth2UserTokenExchange
          Name: ns_incidents_wz_incidents_auth
          ServiceInstanceName: incidents-auth
          ServiceKeyName: incidents-auth-key
          sap.cloud.service: ns.incidents.wz
        ```
    
    4. Add a new destination for the html5 repo runtime under the destination:
        
        ```yaml
        - Name: incident-management_cdm
          ServiceInstanceName: incidents-html5-app-runtime-service
          ServiceKeyName: incidents-html5-app-runtime-service-key
          URL: https://html5-apps-repo-rt.${default-domain}/applications/cdm/<cloud-service-name>

        ```
4. Update the `incident-management-app-content`:
   
   1. Under the `requires` section, the following dependencies:
      
      ```yaml
        requires:
        - name: srv-api
        - name: incidents-auth
      ```
   2. Add the following parameters:
      
      ```yaml
        parameters:
           config:
             destinations:
             - forwardAuthToken: true
               name: capire_incidents-srv-api
               url: ~{srv-api/srv-url}
             - name: ui5
               url: https://ui5.sap.com
      ```

5. Update `incident-management-destination-service` with the following configurations:
  
   1. Under `parameters`, delete `HTML5Runtime_enabled: true`.
   2. Change `instance` to `destination`.
   3. Delete the destinations under it and add the following:
      
      ```yaml
        - Authentication: NoAuthentication
          Name: incident-management-rt
          ProxyType: Internet
          CEP.HTML5contentprovider: true
          Type: HTTP
          URL: https://<tenant_subdomain>-launchpad.${default-domain}
      ```
   
   4. Delete:
        
        ```yaml
        requires:
          - name: srv-api
        ```
6. Update the build parameters if you haven't done this already: 
   
   ```yaml
    build-parameters:
    before-all:
      - builder: custom
        commands:
          - npm ci
          - mkdir -p resources
          - cp workzone/cdm.json resources/cdm.json
          - npx cds build --production    
   ```

## Next Step

[Deploy the Incident Management Application in the SAP BTP, Cloud Foundry Runtime](./3-deploy-to-cf.md)