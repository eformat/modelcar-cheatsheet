# modelcar-cheatsheet

Modelcar's are LLMS in an OCI Image.

- https://github.com/redhat-ai-services/modelcar-catalog

You need a working podman.

Checkout the [modelcar-catalog repo](https://github.com/redhat-ai-services/modelcar-catalog) for more source code, Containerfile examples and modelcar images.

## Make your own modelcar's using ramalama

[RamaLama](https://github.com/containers/ramalama) can do this directly.

```bash
cd ramalama-demo
python3.12 -m venv venv
source venv/bin/activate
pip3 install ramalama
```

or

```bash
cd ramalama-demo
python3.12 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Then simply use `ramalama convert` like so e.g. on a gguf model

Download gguf:

```bash
curl -LO https://huggingface.co/lmstudio-community/DeepSeek-R1-0528-Qwen3-8B-GGUF/resolve/main/DeepSeek-R1-0528-Qwen3-8B-Q4_K_M.gguf
```

Setup

```bash
export QUAY_ORG=eformat
export IMAGE_TAG=latest-ubi
export MODEL_DIR=~/instructlab/models
export MODEL=DeepSeek-R1-0528-Qwen3-8B-Q4_K_M.gguf
```

Convert to an OCI container using `ramalama`

```bash
ramalama convert ${MODEL_DIR}/${MODEL} quay.io/${QUAY_ORG}/${MODEL_lc}:${IMAGE_TAG}
```

## To use in RHOAI

KServe needs a shell, so use a multi-stage image with `ubi-micro` as the deployment follows:

```bash
cat > Containerfile.${MODEL_lc} << EOF
FROM quay.io/${QUAY_ORG}/${MODEL_lc}:latest as builder
FROM registry.access.redhat.com/ubi9/ubi-micro:9.5
COPY --from=builder /models /models
USER 1001
EOF
```

Build + tag, push

```bash
podman build -t quay.io/${QUAY_ORG}/${MODEL_lc}:${IMAGE_TAG} -f Containerfile.${MODEL_lc}
podman push quay.io/${QUAY_ORG}/${MODEL_lc}:${IMAGE_TAG}
```

## Make your own modelcar's using Huggingface

Grab the builder image

```bash
podman pull quay.io/redhat-ai-services/huggingface-modelcar-builder:latest
```

Setup a Containerfile

```bash
export QUAY_ORG=eformat
export IMAGE_TAG=latest-ubi
export MODEL=granite-3.2-2b-instruct
export MODEL_REPO=ibm-granite/${MODEL}
export MODEL_lc=${MODEL,,}

cat > Containerfile.${MODEL_lc} << EOF
FROM quay.io/redhat-ai-services/huggingface-modelcar-builder:latest as base
ENV MODEL_REPO="ibm-granite/granite-3.2-8b-instruct"
RUN python3 download_model.py --model-repo ${MODEL_REPO}
FROM registry.access.redhat.com/ubi9/ubi-micro:9.5
COPY --from=base /models /models
USER 1001
EOF
```

Build + tag, push

```bash
podman build -t quay.io/${QUAY_ORG}/${MODEL_lc}:latest-ubi -f Containerfile.${MODEL_lc}
podman push quay.io/${QUAY_ORG}/${MODEL_lc}:${IMAGE_TAG}
```

## Make your own Model Catalog item

Create the `Third Party` [Model catalog](https://interact.redhat.com/share/8lnAkN6iwWdesATOw3O4) in RHOAI.

```bash
oc -n redhat-ods-applications get configmap model-catalog-unmanaged-sources -o yaml > model-catalog-unmanaged-sources.yaml

wget https://raw.githubusercontent.com/opendatahub-io/model-registry/refs/heads/main/model-catalog/models/third-party-models.yaml
yq -p yaml -o json third-party-models.yaml > third-party-models.json

yq -i '.data.modelCatalogSources = load_str("third-party-models.json")' model-catalog-unmanaged-sources.yaml

-- add the surrounding json manually
    {
      "sources": []
    }

oc apply -f model-catalog-unmanaged-sources.yaml -n redhat-ods-applications
```

Add you own ModelCars !

You can check the supported catalog format using:

```bash
oc -n redhat-ods-applications get configmap model-catalog-sources -o yaml > model-catalog-sources.yaml
```
