![wwh logo](https://cloudwwh.com/wp-content/uploads/2019/10/cropped-logo-3.png)

Using Flux, synchronize a Kubernetes Cluster with Kubernetes Manifests stored in a GitHub Repo. The Development Organization needs to push releases of their application up to Docker Hub so they can be deployed by Flux. Set up a repository with the YAML required to use GitHub Actions Workflow to build the container, tag it, and push it to Docker Hub.
Also, use Flux to promote workloads from Development to Production environments.

Prerequisites:

Create github account https://github.com (cloudwwh)
Create docker hub account https://hub.dokcer.com (cloudwwh)

1. Create a empty repository in docker hub (https://hub.docker.com/r/cloudwwh/gitops)

2. Clone the Project repository and copy the files to your own repository (https://github.com/cloudwwh/gitops)
   Folder Structure under gitops repository https://github.com/cloudwwh/gitops : namespaces  production  python  qa  README.md  workloads 

3. Update the Docker credentials in gitops/.github/workflows/pythonapp.yml file (replace cloudwwh with your Docker hub account)

        echo "${{ secrets.DOCKERPW }}" | docker login -u cloudwwh --password-stdin
        docker image build -t cloudwwh/gitops:hellov1.0 .
        docker push cloudwwh/gitops:hellov1.0

4. Create the Docker password secret in GitHub  Settings - Secrets - Add a new Secret (DOCKERPW)

5. Commit Changes to the file for the Actions (cloudwwh/gitops/python/app.py), Now check the Actions Tab 

6. Github now build the application and pushed to docker hub.

7. We now change the event that fires the workflow from push to pull_request in the Python Application

.github/workflows/pythonapp.yml  -  

on:
  push:
    paths:
    - 'python/*'

change push to pull:

on:
  pull:
    paths:
    - 'python/*'

8. Create a feature branch. Change the Python application, then commit the change. Then create a pull request and merge the change in. Examine the workflow actions run afterward and verify the container was pushed to the Docker Hub Registry.

____________________

QA Version Image is hellov2.0
Production Version Image is hellov1.0


9. Update the QA and Production folder with the hello.yaml relavant images

10. Install the fluxctl software: $ sudo snap install fluxctl --classic

11. Create the flux namespace:  kubectl create namespace flux

12. Set the GHUSER Environment variable to your GitHub username:  export GHUSER=[Your username here]

13. Deploy Flux, to Scann the qa folder:

fluxctl install \
--git-user=${GHUSER} \
--git-email=support@cloudwwh.com \
--git-url=git@github.com:cloudwwh/gitops \
--git-path=namespaces,qa \
--namespace=flux | kubectl apply -f -

14. Set the environment variable for the fluxctl command:  export FLUX_FORWARD_NAMESPACE=flux

15. Grab the RSA key created by Flux:  fluxctl identity

16. Copy and past that RSA key into the Deploy Keys within your GitHub Repo. Use the RSA key copied to the clipboard to add a "Deploy Key" in GitHub, or an SSH RSA Key in other Git servers, to grant read and write access to your QA Kubernetes Cluster.

17. Now since we are scanning in same cluster delete namespaces and scann for production 

Delete the running instance of Flux on your cluster: kubectl delete namespace flux
Delete the deployed hello workload: kubectl delete namespace wwhsample

fluxctl install \
--git-user=${GHUSER} \
--git-email=support@cloudwwh.com \
--git-url=git@github.com:cloudwwh/gitops \
--git-path=namespaces,production \
--namespace=flux | kubectl apply -f -


17. Check the Pods and Deployments

devops@dmanager01:~$ fluxctl -n wwhsample list-workloads

WORKLOAD                    CONTAINER  IMAGE                      RELEASE  POLICY
wwhsample:deployment/hello  hello      cloudwwh/gitops:hellov2.0  ready

devops@dmanager01:~$ kubectl get pods -n wwhsample -o wide

NAME                     READY   STATUS    RESTARTS   AGE     IP       NODE        NOMINATED NODE   READINESS GATES
hello-6fd4cf767c-rrhbl   1/1     Running   0          2m41s   <none>   dworker02   <none>           <none>

devops@dmanager01:~$ kubectl get deployments  -n wwhsample -o wide

NAME    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                      SELECTOR
hello   0/1     1            0           3m20s   hello        cloudwwh/gitops:hellov2.0   app=hello

