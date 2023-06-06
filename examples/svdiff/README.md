# SVDiff
This is an implementation of [SVDiff: Compact Parameter Space for Diffusion Fine-Tuning](https://arxiv.org/abs/2303.11305) by using d🧨ffusers. 



## Running locally with PyTorch

### Installing the dependencies

Before running the scripts, make sure to install the library's training dependencies:

**Important**

To make sure you can successfully run the latest versions of the example scripts, we highly recommend **installing from source** and keeping the install up to date as we update the example scripts frequently and install some example-specific requirements. To do this, execute the following steps in a new virtual environment:
```bash
git clone https://github.com/huggingface/diffusers
cd diffusers
pip install -e .
```

Then cd in the example folder and run
```bash
pip install -r requirements.txt
```

And initialize an [🤗Accelerate](https://github.com/huggingface/accelerate/) environment with:

```bash
accelerate config
```

Or for a default accelerate configuration without answering questions about your environment

```bash
accelerate config default
```

Or if your environment doesn't support an interactive shell e.g. a notebook

```python
from accelerate.utils import write_basic_config
write_basic_config()
```

## Single-Subject Generation
"Single-Subject Generation" is a domain-tuning on a single object or concept (using 3-5 images). For example, you can use [dog toy images](https://github.com/huggingface/diffusers/tree/main/examples/dreambooth#dog-toy-example). 


### Training
According to the paper, the learning rate for SVDiff needs to be 1000 times larger than the lr used for fine-tuning. 

```bash
export MODEL_NAME="runwayml/stable-diffusion-v1-5"
export INSTANCE_DIR="path-to-instance-images"
export CLASS_DIR="path-to-class-images"
export OUTPUT_DIR="path-to-save-model"

accelerate launch train_svdiff.py \
  --pretrained_model_name_or_path=$MODEL_NAME  \
  --instance_data_dir=$INSTANCE_DIR \
  --class_data_dir=$CLASS_DIR \
  --output_dir=$OUTPUT_DIR \
  --with_prior_preservation --prior_loss_weight=1.0 \
  --instance_prompt="photo of a sks dog" \
  --class_prompt="photo of a dog" \
  --resolution=512 \
  --train_batch_size=1 \
  --gradient_accumulation_steps=1 \
  --learning_rate=1e-3 \
  --learning_rate_1d=1e-6 \
  --train_text_encoder \
  --lr_scheduler="constant" \
  --lr_warmup_steps=0 \
  --num_class_images=200 \
  --max_train_steps=500
```

### Inference

After training, SVDiff weights can be loaded in a `set_spectral_shifts` function and you just pass the output models into the original pipeline:
```python
import torch
from transformers import CLIPTextModel
from diffusers import UNet2DConditionModel, StableDiffusionPipeline, DPMSolverMultistepScheduler

from modeling_svdiff import set_spectral_shifts

pretrained_model_name_or_path = "runwayml/stable-diffusion-v1-5"
spectral_shifts_ckpt_dir = "ckpt-dir-path"
device = "cuda" if torch.cuda.is_available() else "cpu"
unet = UNet2DConditionModel.from_pretrained(pretrained_model_name_or_path, subfolder="unet").to(device)
text_encoder = CLIPTextModel.from_pretrained(pretrained_model_name_or_path, subfolder="text_encoder").to(device)
# make sure to move models to device before
unet, _ = set_spectral_shifts(unet, spectral_shifts_ckpt=os.path.join(spectral_shifts_ckpt, "spectral_shifts.safetensors"))
text_encoder, _ = set_spectral_shifts(text_encoder, spectral_shifts_ckpt=os.path.join(spectral_shifts_ckpt, "spectral_shifts_te.safetensors"))
# load pipe
pipe = StableDiffusionPipeline.from_pretrained(
    model_id,
    unet=unet,
    text_encoder=text_encoder,
)
pipe.scheduler = DPMSolverMultistepScheduler.from_config(pipe.scheduler.config)
pipe.to(device)
image = pipe("A picture of a sks dog in a bucket", num_inference_steps=25).images[0]
```

## Single Image Editing
### Training
In Single Image Editing, your instance prompt should be just the description of your input image **without the identifier**. For example, you can download an image from [here](https://unsplash.com/photos/1JJJIHh7-Mk).

```bash
export MODEL_NAME="runwayml/stable-diffusion-v1-5"
export INSTANCE_DIR="dir-path-to-input-image"
export CLASS_DIR="path-to-class-images"
export OUTPUT_DIR="path-to-save-model"

accelerate launch train_svdiff.py \
  --pretrained_model_name_or_path=$MODEL_NAME  \
  --instance_data_dir=$INSTANCE_DIR \
  --class_data_dir=$CLASS_DIR \
  --output_dir=$OUTPUT_DIR \
  --instance_prompt="photo of a pink chair with black legs" \
  --resolution=512 \
  --train_batch_size=1 \
  --gradient_accumulation_steps=1 \
  --learning_rate=1e-3 \
  --learning_rate_1d=1e-6 \
  --train_text_encoder \
  --lr_scheduler="constant" \
  --lr_warmup_steps=0 \
  --max_train_steps=500
```

### Inference
After training, you can load the original pipeline via `set_spectral_shifts` as [Single-Subject Generation](#inference).
```python
import torch
from transformers import CLIPTextModel
from diffusers import UNet2DConditionModel, DDIMScheduler

from modeling_svdiff import set_spectral_shifts

pretrained_model_name_or_path = "runwayml/stable-diffusion-v1-5"
spectral_shifts_ckpt_dir = "ckpt-dir-path"
device = "cuda" if torch.cuda.is_available() else "cpu"
unet = UNet2DConditionModel.from_pretrained(pretrained_model_name_or_path, subfolder="unet").to(device)
text_encoder = CLIPTextModel.from_pretrained(pretrained_model_name_or_path, subfolder="text_encoder").to(device)
# make sure to move models to device before
unet, _ = set_spectral_shifts(unet, spectral_shifts_ckpt=os.path.join(spectral_shifts_ckpt, "spectral_shifts.safetensors"))
text_encoder, _ = set_spectral_shifts(text_encoder, spectral_shifts_ckpt=os.path.join(spectral_shifts_ckpt, "spectral_shifts_te.safetensors"))
# load pipe
pipe = StableDiffusionPipeline.from_pretrained(
    model_id,
    unet=unet,
    text_encoder=text_encoder,
)
pipe.to(device)
```

For Single Image Editing, you can optionally perform DDIM Inversion by using the `ddim_invert` function.
```python
from PIL import Image
from modeling_svdiff import set_spectral_shifts, ddim_invert

image = "path-to-image"
prompt = "prompt-you-want-to-generate"

if image:
  # (optional) ddim inversion
  image = Image.open(image).convert("RGB").resize((512, 512))
  # in SVDiff, they use guidance scale=1 in ddim inversion
  # They use the target prompt rather than source prompt in DDIM inversion for better results 
  inv_latents = ddim_invert(pipe, prompt=prompt, image=image, guidance_scale=1.0)
else:
  inv_latents = None

# They use a small cfg scale in Single Image Editing 
image = pipe(prompt, latents=inv_latents, guidance_scale=3, eta=0.5).images[0]
```

To use slerp to add more stochasticity,
```python
from modeling_svdiff import slerp_tensor

# prev steps omitted
inv_latents = ddim_invert(pipe, prompt=target_prompt, image=image, guidance_scale=1.0)
noise_latents = pipe.prepare_latents(inv_latents.shape[0], inv_latents.shape[1], 512, 512, dtype=inv_latents.dtype, device=pipe.device, generator=torch.Generator("cuda").manual_seed(0))
inv_latents =  slerp_tensor(0.5, inv_latents, noise_latents)
image = pipe(target_prompt, latents=inv_latents).images[0]
```


## Additional Features

### Spectral Shift Scaling

You can adjust the strength of the weights. The larger the scale, the closer the generated image is to your input concept.
On the other hand, decreasing that value generates images that follow the text prompt more.

![scale](https://github.com/mkshing/svdiff-pytorch/raw/main/assets/scale.png)

```python
from modeling_svdiff import set_spectral_shifts
scale = 1.2

unet, _ = set_spectral_shifts(unet, spectral_shifts_ckpt=os.path.join(spectral_shifts_ckpt, "spectral_shifts.safetensors"), scale=scale)
text_encoder, _ = set_spectral_shifts(text_encoder, spectral_shifts_ckpt=os.path.join(spectral_shifts_ckpt, "spectral_shifts_te.safetensors"), scale=scale)
```