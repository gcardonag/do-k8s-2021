# do-k8s-2021
Using the DigitalOcean Kubernetes 2021 Challenge to learn something new! https://www.digitalocean.com/community/pages/kubernetes-challenge

This project sets up Kubeflow and an optional sample deployment of a model that uses the [MNIST dataset](https://en.wikipedia.org/wiki/MNIST_database). The source for the sample can be found here: https://github.com/kubeflow/examples/tree/master/mnist

These steps were performed on system running macOS Big Sur v11.6.2.
# Set up Kubeflow
## Step 1 - Setting up the Kubernetes Cluster
1. Download, install, and configure doctl: https://docs.digitalocean.com/reference/doctl/how-to/install/
1. Run the following command to set up a Kubernetes cluster: `doctl kubernetes cluster create do-k8s-2021 --region nyc3 --set-current-context --wait --version 1.19.15-do.0 --node-pool "name=initial-nodes;size=s-4vcpu-8gb;count=1;auto-scale=true;min-nodes=3;max-nodes=6"`

## Step 2 - Deploying kubeflow
1. Setup kubectl installation dependencies (kustomize):
  1. Run `mkdir bin && export PATH=$PATH:$(pwd)/bin` to create a path to store dependency binaries and make it accessible for later steps.
  1. Download kustomize into bin: `curl -Lo bin/kustomize https://github.com/kubernetes-sigs/kustomize/releases/download/v3.2.0/kustomize_3.2.0_darwin_amd64 && chmod +x bin/kustomize`
    - We download version 3.2.0 per the instructions here: https://github.com/kubeflow/manifests/#prerequisites
1. Setup Kubeflow's v1.4.1 manifest: https://github.com/kubeflow/manifests/tree/v1.4.1
  1. Download the manfifest files: `mkdir -p downloads && curl -Lo downloads/kubeflow_manifests-v1.4.1.tar.gz https://github.com/kubeflow/manifests/archive/refs/tags/v1.4.1.tar.gz`
  1. Unzip the files: `tar -xvzf downloads/kubeflow_manifests-v1.4.1.tar.gz -C downloads && cd downloads/manifests-1.4.1`
1. OPTIONAL: If you would like to change the default password to access kubeflow (recommended if you will be exposing it to the internet), follow the steps detailed here: https://github.com/kubeflow/manifests#change-default-user-password
1. Deploy kubeflow: `while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done`

## Step 3a - Access Kubeflow via Port Forwarding (Recommended)
1. Run `kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80`
1. In a browser, go to `http://localhost:8080/`, a login prompt should appear.
  1. The user/password will depend on the values set in the manifests-1.4.0/common/dex/base/config-map.yaml file and whether the optional password change step was performed.
  1. By default, the user/password combination is user@example.com/12341234
1. You can now use all of kubeflow's components! For more information on how to use kubeflow's components in the UI, visit: https://www.kubeflow.org/docs/components/

## Step 3b - Access Kubeflow via Public Endpoint
NOTE: This step will create a DigitalOcean Load Balancer with 1 node and will incur any costs related to this resource. For more information visit this page: https://www.digitalocean.com/products/load-balancer/
1. Run `kubectl patch service istio-ingressgateway --patch "$(cat istio-ingressgateway_patch.yaml)" -n istio-system`
1. To get the IP of the load balancer, run `kubectl get service istio-ingressgateway -n istio-system --output jsonpath='{.status.loadBalancer.ingress[0].ip}'`
  1. NOTE: This step will not return anything until the load balancer has been deployed and is attached to your cluster. This may take up to 5 minutes. You can see the progress of the load balancer creation in your Digital Ocean console by going to the Networking section and then going to the "Load Balancers" tab.
1. You should now be able to access kubeflow by entering the IP from the previous step in your browser (ex - http://45.55.125.122/)
  1. NOTE: This is an insecure connection to a remote destination with a login flow. For production use, you should consider securing this endpoint behind a domain with HTTPS-based access to encrypt your connections.

# Try using a notebook with Kubeflow's MNIST example
## Step 1 - Set up MNIST example dependencies
1. Run `kubectl apply -f cross-namespace-kubeflow-access.yaml` from the base of this project. This will allow your notebook to access services in the `kubeflow` namespace, such as minio which is used for object storage.
1. Create an access token for Docker Hub and save it: https://docs.docker.com/docker-hub/access-tokens/#create-an-access-token
1. Using the token from the previous step, create a docker config file to apply to docker in a later step: `TOKEN=$(echo -n "YOUR_USERNAME:YOUR_TOKEN"|base64|tr -d \\n) && sed "s/BASE64_USER:PASSWORD/$TOKEN/g" config.json.tpl > config.json`
  1. NOTE: This config file assume you will be uploading images to DockerHub. If you will be using an alternative container image registry, make sure to update the config file appropriately for compatibility with Kaniko.
1. Set up the configmap used to build the model in the notebook `kubectl create --namespace kubeflow-user-example-com configmap docker-config --from-file=config.json`
1. Goto http://localhost:8080/_/jupyter/?ns=kubeflow-user-example-com
1. Click on "New Notebook"
  1. Set the name to any preferred name
  1. Set the image to "j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow-full:v1.4"
1. Once the notebook is ready, click the CONNECT button
1. In the notebook interface, click the Git icon on the lefthand toolbar and then click the "Clone a Repository" button
  1. In the window that appears, enter https://github.com/kubeflow/examples.git and click "CLONE"
  1. Once the cloning is complete, you should see a list of folders on the lefthand toolbar.
1. Open the examples folder, then the mnist folder, then open the `mnist_vanilla_k8s.ipynb` file
1. Click on the notebook section where the "Configure docker credentials" text appears, then click on the `+` symbol on the top toolbar. This should create a new section in the notebook.
1. In this section, paste the following two lines:
```
!pip3 install --quiet -r requirements.txt
!pip3 install --quiet msrestazure
```
1. In the next textbox in the notebook, add the following text to the beginning of the textbox.
```
import logging
from kubernetes import config
config.load_incluster_config()
```
1. In the same textbox, update the line that looks like `DOCKER_REGISTRY = "ciscoai"` to `DOCKER_REGISTRY = "YOUR_DOCKER_USERNAME"`
1. In the same textbox, update the line that looks like `s3_endpoint = mini_service_endpoint` to `s3_endpoint = minio_service_endpoint+":9000"`
1. In the next textbox update the line that looks like `notebook_setup.notebook_setup()` to `notebook_setup.notebook_setup(platform="TBD")`. This is done because the default behavior for the `notebook_setup` function assumes the use of Google Cloud and attempts to look for Google Cloud credentials, leading to an error.
1. Look for the textbox with the text `tfjob_client.get_logs(train_name, namespace=namespace)`, and add a `#` at the beginning of the text. This is done to comment out this step as it consistently fails to deliver logs while testing.
  1. I am open to feedback on how this could be addressed!
1. To start running all of the cells in the notebook, go to "Run" in the top toolbar, then click "Run All Cells". This will start executing all of the cells in the notebook, in order from top to bottom.
1. As the sections of the notebook complete, the `*` symbols to the left of the text box will become a number indicating the order of execution. This step can take 5-10 minutes so its a good opportunity to stretch!
1. If all of the cells show a number next to them, the model should be ready for use via the MNIST UI. To access it, visit the following page: http://localhost:8080/mnist/kubeflow-user-example-com/ui/
  1. This should show a page with an image of number and a prediction of what the number in the image is. There is also a button at the bottom of the page to test against an image of another random number.
1. If you would like to explore the model further, you could access the TensorBoard page: http://localhost:8080/mnist/kubeflow-user-example-com/ui/
  1. If you would like to access it via a public endpoint instead, follow section Step 3b in the previous section and then replace `localhost:8080` in the previous URLs with the public IP collected in that step (ex - 165.227.248.197).

# Cleanup
Since leaving the resources built during this demo will accumulate costs, the following command will help you delete the cluster and the optional Load Balancer created during this demo in Step 3b: `doctl kubernetes cluster delete do-k8s-2021 --dangerous`
