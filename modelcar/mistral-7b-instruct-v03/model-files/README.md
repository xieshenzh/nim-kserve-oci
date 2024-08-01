# Setup

The following steps assumes a running Openshift cluster with RHOAI installed, Kserve enabled, and a NGC account with NIM access.
The cluster should have enough resources to meet computing, storage and GPU requirements for NIM deployment.

1. [Enable](https://kserve.github.io/website/latest/modelserving/storage/oci/#enabling-modelcars) the `modelcar` feature for Kserve on the cluster.
    - Create a manifest for `inferenceservice-config` ConfigMap.
    - Add or modify `enableModelcar` to `true` in `storageInitializer`. (See [inferenceservice.yaml](./rhoai/inferenceservice.yaml%20) as an example)
    - Commit the change and push to a git repo.
    - Change the `kserve` configuration in `DataScienceCluster` CR to use the manifest. For example:
   ```yaml
   kserve:
     managementState: Managed
     serving:
       ingressGateway:
         certificate:
          type: SelfSigned
       managementState: Managed
       name: knative-serving
     devFlags:
       manifests: 
       - uri: https://github.com/github/xieshenzh/nim-kserve-oci/refs/heads/main.tar.gz
         contextDir: modelcar
         sourcePath: model-files/rhoai
   ```
    - Check `inferenceservice-config` ConfigMap in the `redhat-ods-applications` Project if `enableModelcar` is set to `true`.
    - Restart the `kserve-controller-manager` Pod in the `redhat-ods-applications` Project.
2. Based on the [guild](https://github.com/NVIDIA/nim-deploy/blob/main/kserve/README.md) and [scripts](https://github.com/NVIDIA/nim-deploy/blob/main/kserve/scripts/README.md), make necessary changes to the Openshift cluster and make it ready to deploy NIM.
3. Create a Project to deploy NIM on the cluster.
4. Create a image pulling Secret in the Project for pulling NIM images.
   ```shell
   oc create secret docker-registry ngc-secret \
      --docker-server=nvcr.io\
      --docker-username='$oauthtoken'\
      --docker-password=${NGC_API_KEY}
   ```
5. Create a Secret in the Project for accessing NIM in the Pods using the [manifest](./download/kserve/nvidia-nim-secrets.yaml). Make sure the `NGC_API_KEY` is properly set in the manifest.
6. Create NIM download cache by deploying NIM in Openshift.
    1. Create the ServingRuntime CR for NIM deployment with the [manifest](./download/kserve/mistral-7b-instruct-v03-1.0.0.yaml).
    2. Create the InferenceService CR to deployment NIM with the [manifest](./download/kserve/mistral-7b-instruct-v03_1xgpu_1.0.0.yaml). Adjust the `nodeSelector` and `tolerations` configurations based on the cluster resources and settings.
    3. Check the status of the InferenceService CR, wait until it is ready to serve.
   ```shell
   oc get inferenceservice mistral-7b-instruct-v03-1xgpu 
   ```
7. Create a model repository in the Pod.
   1. Find the name of the pod for NIM.
   ```shell
   oc get pods
   ```
   2. Get a shell to the running container of NIM.
   ```shell
   oc exec --stdin --tty <pod name> -c kserve-container -- /bin/bash
   ```
   3. In the shell, use the [list-model-profiles](https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html#serving-models-from-local-assets) command to list the available profiles.
   ```shell
   list-model-profiles
   ```
   4. Select a compatible profile, use the `create-model-store` command to create a repository for the model.
   ```shell
   create-model-store --profile <profile> --model-store /tmp/model
   ```
   5. Close the shell.
8. Change to this directory. Download the NIM model repository to local.
    1. Find the name of the pod for NIM.
   ```shell
   oc get pods
   ```
    2. Copy the download cache to local. For example:
   ```shell
   oc rsync <pod name>:/tmp/model ./download -c kserve-container
   ```
9. Build an OCI image with the NIM model files with the Dockerfile, and push to registry. For example:
   ```shell
   podman build . -f ./docker/Dockerfile -t quay.io/xiezhang7/mistral-7b-instruct-v03:v1.0.0
   ```
10. Delete the InferenceService and ServingRuntime CRs for generating the NIM download cache from the Project.
   ```shell
   oc delete InferenceService mistral-7b-instruct-v03-1xgpu
   oc delete ServingRuntime nvidia-nim-mistral-7b-instruct-v03-1.0.0
   ```
11. Create the ServingRuntime CR for NIM deployment with the [manifest](./kserve/mistral-7b-instruct-v03-1.0.0.yaml).
12. Create the InferenceService CR to deployment NIM with the [manifest](./kserve/mistral-7b-instruct-v03_1xgpu_1.0.0.yaml). Adjust the `nodeSelector` and `tolerations` configurations based on the cluster resources and settings. Change the `storageUri` value to your NIM model cache OCI image name.
13. Check the status of the InferenceService CR, and get its endpoint URL when it is ready.
   ```shell
   oc get inferenceservice mistral-7b-instruct-v03-1xgpu
   ```
14. Validate the NIM deployment by posting a query.
   ```shell
   curl --insecure ${URL}/v1/models
   ```
   ```shell
   curl --insecure ${URL}/v1/chat/completions \                                                                 ─╯
     -H "Content-Type: application/json" \
     -d '{
       "model": "/mnt/models/",
       "messages": [{"role":"user","content":"What is KServe?"}],
       "temperature": 0.5,
       "top_p": 1,
       "max_tokens": 1024,
       "stream": false
       }'
   ```