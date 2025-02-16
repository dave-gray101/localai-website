
+++
disableToc = false
title = "Easy Setup - GPU Docker"
weight = 2
+++

We are going to run LocalAI with `docker-compose` for this set up.


Lets clone LocalAI with git.

```bash
git clone https://github.com/go-skynet/LocalAI
```


Then we will cd into the LocalAI folder.

```bash
cd LocalAI
```


At this point we want to set up our `.env` file, here is a copy for you to use if you wish, please make sure to set it to the same as the docker-compose file for later.

```bash
## Set number of threads.
## Note: prefer the number of physical cores. Overbooking the CPU degrades performance notably.
THREADS=2

## Specify a different bind address (defaults to ":8080")
# ADDRESS=127.0.0.1:8080

## Default models context size
# CONTEXT_SIZE=512
#
## Define galleries.
## models will to install will be visible in `/models/available`
GALLERIES=[{"name":"model-gallery", "url":"github:go-skynet/model-gallery/index.yaml"}, {"url": "github:go-skynet/model-gallery/huggingface.yaml","name":"huggingface"}]

## CORS settings
# CORS=true
# CORS_ALLOW_ORIGINS=*

## Default path for models
#
MODELS_PATH=/models

## Enable debug mode
DEBUG=true

## Specify a build type. Available: cublas, openblas, clblas.
BUILD_TYPE=cublas

## Uncomment and set to true to enable rebuilding from source
REBUILD=true

## Enable go tags, available: stablediffusion, tts
## stablediffusion: image generation with stablediffusion
## tts: enables text-to-speech with go-piper 
## (requires REBUILD=true)
#
#GO_TAGS=tts

## Path where to store generated images
# IMAGE_PATH=/tmp

## Specify a default upload limit in MB (whisper)
# UPLOAD_LIMIT
# HUGGINGFACEHUB_API_TOKEN=Token here
```


Now that we have the .env set lets set up our docker-compose file, this docker-compose file is for CUDA:

```docker
version: '3.6'

services:
  api:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    image: quay.io/go-skynet/local-ai:master-cublas-cuda12
    tty: true # enable colorized logs
    restart: always # should this be on-failure ?
    ports:
      - 8080:8080
    env_file:
      - .env
    volumes:
      - ./models:/models
    command: ["/usr/bin/local-ai" ]
```


Make sure to save that in the root of the LocalAI folder. Then lets spin up the docker run this in a CMD or BASH

```bash
docker-compose up -d --pull always
```


Now we are going to let that set up, once it is done, lets check to make sure our huggingface / localai galleries are working (wait until you see the ready screen to do this)

```bash
curl http://localhost:8080/models/available
```

Output will look like this:

![](https://cdn.discordapp.com/attachments/1116933141895053322/1134037542845566976/image.png)

Now lets pick a model to download and test out. We are going to use WizardLM-13B-V1.2-GGML, there are a few ways to do this, 

Way number one is old school, download and move the model.

Link - https://huggingface.co/TheBloke/WizardLM-13B-V1.2-GGML

Using that link download the wizardlm-13b-v1.2.ggmlv3.q4_0.bin model, once done, move the model.bin into the models folder.

Way number two is the galleries. In the docker cmd run this.
```bash
curl --location 'http://localhost:8080/models/apply' \
--header 'Content-Type: application/json' \
--data-raw '{
    "id": "TheBloke/wizardlm-13b-v1.2-ggml/wizardlm-13b-v1.2.ggmlv3.q4_0.bin",
    "name": "lunademo"
}'
```

Now lets make 3 files.

```bash
touch wizardlm-chat.tmpl
touch wizardlm-completion.tmpl
touch lunademo.yaml
```

Please note the names for later!

In the "wizardlm-chat.tmpl" file add

```txt
{{.Input}}

### Response:
```

In the "wizardlm-completion.tmpl" file add

```txt
Complete the following sentence: {{.Input}}
```


In the "lunademo.yaml" file

```yaml
backend: llama
context_size: 2000
f16: true 
gpu_layers: 4
low_vram: true
mmap: true
mmlock: false
batch: 512
name: lunademo
parameters:
  model: wizardlm-13b-v1.2.ggmlv3.q4_0.bin
  temperature: 0.2
  top_k: 40
  top_p: 0.65
roles:
  assistant: '### Response:'
  system: '### System:'
  user: '### Instruction:'
template:
  chat: wizardlm-chat
  completion: wizardlm-completion
```

Now that we have that fully set up, we need to reboot the docker. Go back to the localai folder and run

```bash
docker-compose restart
```


Now we can make openai / curl request!

Curl - 

```bash
curl http://localhost:8080/v1/chat/completions -H "Content-Type: application/json" -d '{
     "model": "lunademo",
     "messages": [{"role": "user", "content": "How are you?"}],
     "temperature": 0.9 
   }'
```


OpenAI Python -

```python
import os
import openai
openai.api_base = "http://localhost:8080/v1"
openai.api_key = "sx-xxx"
OPENAI_API_KEY = "sx-xxx"
os.environ['OPENAI_API_KEY'] = OPENAI_API_KEY

completion = openai.ChatCompletion.create(
  model="lunademo",
  messages=[
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "How are you?"}
  ]
)

print(completion.choices[0].message)
```

Have fun using LocalAI!
