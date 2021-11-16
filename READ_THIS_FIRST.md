## Generated Artifacts
| Artifact | Description |
| --- | --- |
| modresorts-1.0.war | Application binary ear, or war or jar file |
| Dockerfile | Dockerfile to build the application container image |
| src/config/install_app.py | Jython script to install the application |
| src/config/server_config.py | server configuration Jython script file |  
| lib/ | Directory for external library dependencies, such as database driver library | 
| pipeline/k8s/ | Directory for the application deployment manifest files |
| pipeline/openshift/ | Directory for OpenShift pipeline manifest files |

### Notes
- The pipeline artifacts for openshift are compatible with OpenShift 4.6 and 4.7.

- The generated `src/config/server_config.py` script file may contain password variable but its value has been stripped off from
the original source environment.  You need to provide it manually. For example, 

  ```
  # The following variables are used to replace sensitive data in the configuration for the application.
  # The values for these variables were not collected because the includeSensitiveData option was not specified.
  # ============================================================
  myhost_a1CellManager01_db2_password_1=''
  # ============================================================  
  ```

## Deploy to Red Hat OpenShift Container Platform using OpenShift Pipelines

### Prerequisites
1. Install the OpenShift Pipeline Operator if its not already installed. See OpenShift [documentation](https://docs.openshift.com/container-platform/4.6/pipelines/installing-pipelines.html) for more details.

2. An available `PersistentVolume` with at least 100Mi capacity and `accessMode` of `ReadWriteOnce`. The pipeline will create and use a `PersistentVolumeClaim` that will bind to this available `PersistentVolume`.
   The storage should have appropriate permissions to allow writing from the pipeline pods.

3. If your git repository is private, then you will need to set authentication. See "Basic Authentication for Git" section below.

4. If you wish to control and monitor your pipeline from the command line, install the OpenShift pipeline CLI by following the instructions [here](https://github.com/tektoncd/cli) 

### Steps
1. Login to OCP cluster and create a project
   ```
   oc login --token={your_openshift_cluster_access_token} --server=your_openshift_cluster_url:6443
   oc project myproject
   ```

2. Clone this git repository with the migration artifacts to your local machine and cd to the repository root.

3. Create a OpenShift pipeline and pipeline resources for building and deploying this application

   - Update the image url value in `03-pipeline-run.yaml` if you are deploying the application to a 
   different project. i.e. change `myproject` in the following line to match your namespace.
   ```
     params:
     - name: image-url
       value: image-registry.openshift-image-registry.svc:5000/myproject/application:latest
   ```
   
   - The branch for the git repo is set to `master` by default, if you wish to update that please do so in the `03-pipeline-run.yaml` file.
   
   - Set context root in route resource in the file `pipeline/k8s/route.yaml`. The path is set to / by default.
   
   - Push any changes you have made to your remote repo.
   
   - Create the resources and start a pipeline run:
   ```
   oc create -f pipeline/openshift
   ```
   - Check that the `pvc` was created and bound successfully:
   ```
   oc get pvc
   ```
   If that command does not show a bound `pvc` called `shared-task-storage`, check your persistence configuration.
   
  
4. After the pipeline run has successfully finished, find the application route from Openshift dashboard or the following command.
   
   ```
   oc get routes
   ```


### Basic Authentication for Git

Define a secret containing the username and access token to the github repository if the repository is a private repository or GitHub enterprise repository.

- create a secret.yaml file and replace the username and password value
  ```
  apiVersion: v1
  kind: Secret
  type: kubernetes.io/basic-auth
  metadata:
    annotations:
      tekton.dev/git-0: https://github.com   # or change to your GHE url, e.g. https://github.enterprise.com
    name: ghe-token
    labels:
      serviceAccount: pipeline
  data:
    password: base64_encoded_github_access_token
    username: base64_encode_github_user_id
  ```
- create the github authentication secret
  ```
  oc apply -f secret.yaml 
  ```
- link the secret to the `pipeline` service account
  ```
  oc secrets link pipeline ghe-token
  ```
- If running the pipeline from the command line, start the Tekton pipeline run with the `pipeline` service account
  ```
  tkn pipeline start build-and-deploy -s pipeline
  ```

### References

https://github.com/WASdev/ci.docker.websphere-traditional#readme

https://github.com/WASdev/ci.docker.websphere-traditional/tree/master/samples/hello-world/openshift
