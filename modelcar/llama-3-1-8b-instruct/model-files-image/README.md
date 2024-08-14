# Setup

1. [Set up](https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html#setup) NIM environment. Install CUDA Drivers and NVIDIA Container Toolkit.

2. [Configure](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuration) NVIDIA Container Toolkit for Podman and check if the configuration is successful (e.g. [running with CDI](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html#running-a-workload-with-cdi)).

3. In the terminal, [log in](https://docs.nvidia.com/launchpad/ai/base-command-coe/latest/bc-coe-docker-basics-step-02.html#logging-in-to-ngc-on-a-workstation) to `ngc` with Podman using an API Key that can access NIM artifacts.
   ```shell
   $ podman login nvcr.io
   Username: $oauthtoken
   Password: <API Key>
   ```
4. Change to this directory, use the [list-model-profiles](https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html#serving-models-from-local-assets) command to list the available profiles.
   ```shell
   podman run --rm --platform=linux/amd64 nvcr.io/nim/meta/llama-3.1-8b-instruct:1.1.1 list-model-profiles
   ```
   - Note: Apply Podman options/flags based on the NVIDIA Container Toolkit configuration, and enable CUDA drivers and GPU access in the container. For example, using Podman on RHEL 9.4 with CDI:
   ```shell
   podman run --rm --platform=linux/amd64 --device nvidia.com/gpu=all --security-opt=label=disable nvcr.io/nim/mistralai/mistral-7b-instruct-v03:1.0.0 list-model-profiles
   ```
5. Select a compatible profile, use the `create-model-store` command to create a model repository and store it to the `download` directory.
   ```shell
   podman run --rm --platform=linux/amd64 -e NGC_API_KEY=<API Key> -v ./download:/opt/nim/.cache/model-store nvcr.io/nim/meta/llama-3.1-8b-instruct:1.1.1 create-model-store --profile <profile> --model-store /opt/nim/.cache/model-store
   ```
   - Note: Apply Podman options/flags based on the NVIDIA Container Toolkit configuration, and enable CUDA drivers and GPU access in the container. For example, using Podman on RHEL 9.4 with CDI:
   ```shell
   podman run --rm --platform=linux/amd64 --device nvidia.com/gpu=all --security-opt=label=disable -e NGC_API_KEY=<API Key> -v ./download:/opt/nim/.cache/model-store nvcr.io/nim/mistralai/mistral-7b-instruct-v03:1.0.0 create-model-store --profile <profile> --model-store /opt/nim/.cache/model-store
   ```
6. Build an OCI image with the model repository using the Dockerfile, and push to image registry. For example:
   ```shell
   podman build . -f ./docker/Dockerfile -t quay.io/xiezhang7/nim-meta-llama-3.1-8b-instruct:v1.1.1-<profile>
   ```
7. Follow [steps](../../mistral-7b-instruct-v03/model-files) 1, 2, 3, 4, 5, 11, 12 and 13 to deploy the model. Make necessary modifications to the manifests, and apply the correct model, image, resources and property names that matches the model repository image created in this PoC.