# Setup

1. In the terminal, [log in](https://docs.nvidia.com/launchpad/ai/base-command-coe/latest/bc-coe-docker-basics-step-02.html#logging-in-to-ngc-on-a-workstation) to `ngc` with Podman using an API Key that can access NIM artifacts.
   ```shell
   $ podman login nvcr.io
   Username: $oauthtoken
   Password: <API Key>
   ```
2. Change to this directory, use the [list-model-profiles](https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html#serving-models-from-local-assets) command to list the available profiles.
   ```shell
   podman run --rm --platform=linux/amd64 nvcr.io/nim/meta/llama-3.1-8b-instruct:1.1.1 list-model-profiles
   ```
3. Select a compatible profile, use the `create-model-store` command to create a model repository and store it to the `download` directory.
   ```shell
   podman run --rm --platform=linux/amd64 -e NGC_API_KEY=<API Key> -v ./download:/opt/nim/.cache/model-store nvcr.io/nim/meta/llama-3.1-8b-instruct:1.1.1 create-model-store --profile <profile> --model-store /opt/nim/.cache/model-store
   ```
4. Build an OCI image with the model repository using the Dockerfile, and push to image registry. For example:
   ```shell
   podman build . -f ./docker/Dockerfile -t quay.io/xiezhang7/nim-meta-llama-3.1-8b-instruct:v1.1.1-<profile>
   ```
5. Follow [steps](../../mistral-7b-instruct-v03/model-files) 1, 2, 3, 4, 5, 11, 12 and 13 to deploy the model. Make necessary modifications to the manifests, and apply the correct model, image, resources and property names that matches the model repository image created in this PoC.