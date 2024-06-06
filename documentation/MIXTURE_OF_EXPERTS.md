# Mixture-of-Experts

SimpleTuner allows splitting the task of training in two, such that the self-attention and cross-attention stages of inference can effectively be split between two entirely different sets of weights.

In this example, we will use SegMind's collaborative effort with Hugging Face, [SSD-1B](https://huggingface.co/segmind/ssd-1b) to create two new models that train more reliably and have better resulting fine details than a single model.

Thanks to the small size of the SSD-1B model, training on even lighter-weight hardware is possible. Since we're starting our model from their pretrained weights, we have to abide by their Apache 2.0 license - but this is relatively straightforward. You can even use the resulting weights in a commercial setting!

When SDXL 0.9 and 1.0 were introduced, they both contained a full base model with a split-schedule refiner.

- The base model was trained on steps 999 to 0
  - The base model is more than 3B parameters, and functions entirely standalone.
- The refiner model was trained on steps 199 to 0
  - The refiner model is also more than 3B parameters, a seemingly unnecessary waste of resources. It does not function on its own without substantial deformations and a bias toward cartoonish outputs.

Let's see how we can improve this situation.


## The Base model, "Stage One"

The first portion of a mixture-of-experts is known as the base model. As mentioned in SDXL's case, it is trained on all 1000 timesteps - but it doesn't need to be. The following configuration will train the base model on just 650 steps out of the total 1000, saving time and training more reliably.

### Environment configuration

Setting the following values in your `sdxl-env.sh` will get us started:

```bash
# Ensure these aren't incorrectly set.
export USE_BITFIT=false
export USE_DORA=false
# lora could be used here instead, but the concept hasn't been explored.
export MODEL_TYPE="full"
export MODEL_NAME="segmind/SSD-1B"
export LEARNING_RATE=4e-7

# We really want this as high as you can tolerate.
# If training is very slow, ensure you alter your CHECKPOINT_STEPS and VALIDATION_STEPS are set low enough that you'll get a checkpoint in an hour or two.
export TRAIN_BATCH_SIZE=8
export GRADIENT_ACCUMULATION_STEPS=4

# If you are running on a beefy machine that doesn't fully utilise its VRAM during training, set this to "false" and your training will go faster.
export USE_GRADIENT_CHECKPOINTING=true

# Enable first stage model training
export TRAINER_EXTRA_ARGS="--refiner_training --refiner_training_strength=0.35 --refiner_training_invert_schedule"

# Optionally reparameterise it to v-prediction/zero-terminal SNR. 'sample' prediction_type can be used instead for x-prediction.
# This will start out looking pretty terrible until about 1500-2500 steps have passed, but it could be very worthwhile.
export TRAINER_EXTRA_ARGS="${TRAINER_EXTRA_ARGS} --prediction_type=v_prediction --rescale_betas_zero_snr --training_scheduler_timestep_spacing=trailing"
```

### Dataloader configuration

No special considerations for dataloader configuration are necessary. See the [dataloader config guide](/documentation/DATALOADER.md) for more information on this step.

### Validation

Currently, SimpleTuner does not engage the second stage model during stage one evaluations.

Future work will support this as an option, in case a stage two model already exists, or is being trained concurrently.

---

## The Refiner model, "Stage Two"

### Comparison to training the SDXL refiner

- Unlike the SDXL refiner, when using Segmind SSD-1B for both stages the text embeds **can** be shared between the two training jobs
  - The SDXL refiner uses a different text embed layout versus the SDXL base model.
- The VAE embeds **can** be shared between the training jobs, just like the SDXL refiner. Both models use the same input layout.
- No aesthetic score is used for the Segmind models, instead they use the same microconditioning inputs as SDXL, eg. crop coordinates
- Training goes much faster, as the model is much smaller, and text embeds can be reused from stage one training

### Environment Configuration

Update the following values in your `sdxl-env.sh` to swap training over to your stage two model. It might be convenient to keep a copy of the base model configuration.

```bash
# Update your OUTPUT_DIR value, so that we don't overwrite the stage one model checkpoints.
export OUTPUT_DIR="/some/new/path"

# We'll swap --refiner_training_invert_schedule for --validation_using_datasets
# - Train the end of the model instead of the beginning
# - Validate using images as input for partial denoising to evaluate fine detail improvements
export TRAINER_EXTRA_ARGS="--refiner_training --refiner_training_strength=0.35 --validation_using_datasets"

# Don't update these values if you've set them on the stage one. Be sure to use the same parameterisation for both models!
export TRAINER_EXTRA_ARGS="${TRAINER_EXTRA_ARGS} --prediction_type=v_prediction --rescale_betas_zero_snr --training_scheduler_timestep_spacing=trailing"
```

### Dataset format

The images should be purely high-quality - remove any datasets you find questionable in terms of compression artifacts or other errors.

Other than that, the same exact dataloader configuration can be used between the two training jobs.

If you'd like a demonstration dataset, [pseudo-camera-10k](https://huggingface.co/datasets/ptx0/pseudo-camera-10k) is a solid choice with permissive licensing.

### Validation

Stage two refiner training will automatically select images from each of your training sets, and use those as inputs for partial denoising at validation time.

## Putting it all together at inference time

If you'd like to plug both of the models together to experiment with in a simple script, this will get you started:

```py
from diffusers import StableDiffusionXLPipeline, StableDiffusionXLImg2ImgPipeline
from torch import float16

stage_1_model_id = 'ptx0/terminus-xl-velocity-v2'
stage_2_model_id = 'ptx0/terminus-xl-refiner'
torch_device = 'cuda' if torch.cuda.is_available() else 'mps' if torch.backends.mps.is_available() else 'cpu'

pipe = StableDiffusionXLPipeline.from_pretrained(stage_1_model_id, add_watermarker=False, torch_dtype=float16).to(torch_device)
img2img_pipe = StableDiffusionXLImg2ImgPipeline.from_pretrained(stage_2_model_id).to(device=torch_device, dtype=float16)

prompt = "An astronaut riding a green horse"

# Important: update this to True if you reparameterised the models.
use_zsnr = False

image = pipe(
    prompt=prompt,
    num_inference_steps=40,
    denoising_end=0.8,
    guidance_scale=9.2,
    guidance_rescale=0.7 if use_zsnr else 0.0,
    output_type="latent",
).images
image = img2img_pipe(
    prompt=prompt,
    num_inference_steps=40,
    denoising_start=0.8,
    guidance_scale=9.2,
    guidance_rescale=0.7 if use_zsnr else 0.0,
    image=image,
).images[0]
image.save('demo.png', format="PNG")

```