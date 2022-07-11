# Gopaddle-Github-Actions


Continuous Integration (CI) and Rolling Update to a kubernetes service can be achieved seamlessly when using gopaddle Deck and Propeller together. gopaddle Deck does an automatic source to image conversion and uses Kubernetes based build environments to build Docker images. gopaddle Propeller helps to create services and application templates that can be deployed across different kubernetes environments. You can enable Continuous Integration in Deck, such that a Docker build is triggered as soon as a code commit or a code merge happens in github. When Deck and Propeller are used together, a seamless CI/CD can be achieved with a robust version controlling across the kubernetes resources. However, in some scenarios, you may choose to use a different tool/platform in place of Deck to build the docker images and use Propeller deploying and managing the applications. For example, you may choose to use github Actions and then integrate with Propeller for rolling updates. Here is a comparison of github Actions vs Deck and when one can be used instead of the other. Irrespective of the build tool used, gopaddle exposes APIs that can be integrated with any third party tools.

github Actions	gopaddle Deck
Dockerfile	Must pre-exist in the github repo.	Auto-generate Dockerfile or use an existing Dockerfile
Build Environment	* On-demand Virtual Hosted environments or Pre-provisioned self-hosted environments. Self hosted environments can be physical or virtual machines or containers.
* Requires configuration effort to setup self-hosted environments
* Runs on VMs or on Containers. Scaling up/down the environments requires requires automation.	* On-demand: Runs inside your kubernetes cluster as isolated build jobs with a specified CPU/Mem capacity.
* Requires no additional effort.
* Uses shared kubernetes environment and can save cost as the environments scales up/down automatically based on build jobs.
Shared Environments	Can be dedicated for a single repo or shared within/ across organizations	Can be run on a shared kubernetes cluster, but new jobs are created for every build.
OS support for build environments	Linux, Windows and MacOS. For complete list of OS supported check here.	Bring your own OS image. Any Linux Operating system.
Architectures	x64 – Linux, macOS, Windows.
ARM64 – Linux only.
ARM32 – Linux only.	Intel x86_64  – Linux only.
AMD64 – Linux only.
ARM64 – Linux only.
Multi-arch – Linux only.
Subscription	Hosted environments has free minutes beyond which per-minute rates are applied. Self-hosted environments are maintained by end-users on their local infrastructure.	Billed based on build concurrency and total number of builds.
github Actions vs gopaddle Deck
If you choose to use github actions for building the containers, then this blog covers the method of integrating github Actions with gopaddle propeller.

Here are some simple steps to follow:

Create Github Actions
Register Docker Registry in gopaddle
Create an Image based container in gopaddle
Create necessary artifacts in gopaddle and deploy the service on a Kubernetes environment
Generate gopaddle API token
Update github actions to invoke gopaddle API
Let us explore each of these steps in detail.


Trigger Rolling Update directly from github actions
Create Github Actions

As the first step, we will set the github actions to build a docker container. You can use github secrets to store sensitive information like credentials and tokens. Then create a github workflow using those secrets. We will be using Azure Docker Registry to push the docker image. Hence we will use the azure/docker-login@v1 plugin in the workflow.

Let us create the secrets in github repository. Under settings option in github respository, edit Secrets and Add a New respository secret. We will add the Azure credentials in the github secret for the time being. But as we progress through the remaining steps, we will need to add the secrets for gopaddle API token as well. You can follow the steps documented under Azure ACR Registry section here to create an Azure Registry. Here is the list of secrets we need in order to use Azure Registry. You will need these information while registering the Azure Registry in gopaddle as well.
REGISTRY_LOGIN_SERVER
REGISTRY_USERNAME
REGISTRY_PASSWORD

github repository secrets
2. Now let us create the .github/workflows folder in the repository. Go to the workflow tab and create the yaml file to build your source code. Your github actions task would look like this to build and push the image to the docker registry. petclinic is the name of the docker image and we will use the github commit ID as the docker image tag. By using the commit ID as the tag, we can relate the docker image to the commit that triggered the build for future debugging.

    - name: 'Build and push image'
      uses: azure/docker-login@v1
      with:
        login-server: ${{ secrets.REGISTRY_LOGIN_SERVER }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    - run: |
            docker build . -t ${{ secrets.REGISTRY_LOGIN_SERVER }}/petclinic:${{ github.sha }}
            docker push ${{ secrets.REGISTRY_LOGIN_SERVER }}/petclinic:${{ github.sha }}
3. Execute github actions once, so that a docker image is created and pushed to docker registry. We will be using this image in the next steps.

Register Docker Registry in gopaddle
We will be using the image created in earlier step to deploy an application in gopaddle. Once an application is deployed, we can perform further rolling updates on that application. We can automate this initial deployment process as well using gopaddle APIs. But for the time being, we will manually deploy an application using gopaddle UI. To create the gopaddle artifacts, Docker Registry has to be registered with gopaddle, so that gopaddle knows the source from which a docker image can be pulled and deployed.

4. Register the Azure Registry in gopaddle as described here.

Create necessary artifacts in gopaddle and deploy the service on a Kubernetes environment
5. We need to create 3 resources in gopaddle.

(a) Container : Create an image based container in gopaddle using the image created in the earlier step as described here. Once the container is created, note down the container ID by viewing the container and extracting the ID from the browser URL.


(b) Service : Create a Service and add the container to the service as described here. Once the service is created, note down the service ID by viewing the service and extracting the ID from the browser URL.


(c) Deployment Template : Create a Deployment Template and add the Service to the template.

6. Create a Kubernetes cluster as described here and deploy the template on the Kubernetes cluster to create a running application.

7. Once the application is launched, view the application and gather the application ID from the browser URL.


Using the container ID, service ID and the application ID, we can trigger the rolling update on the specific container within an application as soon as the build is complete.

Generate gopaddle API token
gopaddle APIs can be securely invoked by using an API token. A user can generate one or more API tokens. A role defines the permissions for a user. A role can be assigned to a user to restrict actions within gopaddle. Let us first create a role, a user and then assign the role to the user. Once the user is created, we will generate an API token.

8. In the gopaddle portal, under user profile, navigate to Teams -> Access Control Lists and create a new role with access to All Services and All Permissions for the time being.


gopaddle – create a role with All Permissions
12. Under the Users tab, create a user and assign the role.

13. Click on User actions and Add a token


Update github actions to invoke gopaddle API
Now we have the necessary details to invoke gopaddle APIs.

Container ID
Service ID
Application ID
API Token
14. Add this API token in the github secrets.

15. Update the github actions code to invoke gopaddle API as below.

Your github actions YAML would look like this: We will be using fjogeleit/http-request-action@master github plugin to trigger an API call.

