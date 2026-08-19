---
layout: page
title: 
permalink: /research/thesis/
---
<h3></h3>

## Leverage LLM to build personal assistant

**Date:** 2023\
**Paper:** [https://ieeexplore.ieee.org/document/10288647](https://ieeexplore.ieee.org/document/10288647)\
**Github:** [https://github.com/doanthuan/bloom-gpt](https://github.com/doanthuan/bloom-gpt)

### Overview
- Build general chatbot and doctor assistant based on LLM.
- Create high quality instruction data with 200k samples.
- Finetune Bloomz model using LoRA technique

![Image](/assets/thesis/overal.png)

### Data
- Adopt Self-Instruct technique with GPT-4 to augment dataset
- Collect 200k real conversations between patients and doctors from [Health Care Magic](healthcaremagic.com)

<img src="/assets/thesis/data3.png" width="600"/>

<img src="/assets/thesis/data4.png" width="600"/>

<img src="/assets/thesis/data1.png" width="600"/>

<img src="/assets/thesis/data2.png" width="400"/>





### Finetuning

- Pretrained model: Bloomz-mt-7B
- Finetuning technique: LoRA

<img src="/assets/thesis/finetune1.png" width="600"/>

- Training details:
    - Libraries: PyTorch and Huggingface Transformers.
    - context length: 512 and the rank k in LoRA to 8. 
    - 8-bit integer format (int8) parameters released by
    - Adam optimizer to update LoRA parameters with a total batch size of 128 and learning rates of 3e-4.
    - The trainable LoRA parameters are about 4.2M parameters and 
    - Fine-tuned for 2 epochs on 4 RTX-4090-24GB GPU

### Setup Enviroment

1. Install dependencies

   ```bash
   pip install -r requirements.txt
   ```

1. If bitsandbytes doesn't work, [install it from source.](https://github.com/TimDettmers/bitsandbytes/blob/main/compile_from_source.md) Windows users can follow [these instructions](https://github.com/tloen/alpaca-lora/issues/17).

### Download Dataset

- `instruct_merged.jsonl`: instruction dataset. It contains 52k samples from Alpaca + 170k samples from GPT4All. Then translated to Vietnamese.

   ```bash
   wget https://storage.googleapis.com/doanthuan/data/instruct_merged.jsonl
   ```

- `translated_health_200k.jsonl`: Medical instruction dataset. It was collected from [ChatDoctor](https://github.com/Kent0n-Li/ChatDoctor)

   ```bash
   wget https://storage.googleapis.com/doanthuan/data/translated_health_200k.jsonl
   ```


### Training

Finetune Bloomz-Chat:

```bash
python finetune_bloomz_instruct.py \
    --base_model 'bigscience/bloomz-7b1-mt' \
    --data_path 'instruct_merged.jsonl' \
    --output_dir './bloomz-instruct'
```

Finetune Bloomz-Doctor:

```bash
python finetune_bloomz_doctor.py \
    --base_model 'bigscience/bloomz-7b1-mt' \
    --data_path 'translated_health_200k.jsonl' \
    --output_dir './bloomz-doctor'
```

We can also tweak our hyperparameters:

```bash
python finetune_bloomz_instruct.py \
    --base_model 'bigscience/bloomz-7b1-mt' \
    --data_path 'instruct_merged.jsonl' \
    --output_dir './bloomz-instruct' \
    --batch_size 128 \
    --micro_batch_size 4 \
    --num_epochs 3 \
    --learning_rate 1e-4 \
    --cutoff_len 512 \
    --val_set_size 2000 \
    --lora_r 8 \
    --lora_alpha 16 \
    --lora_dropout 0.05 \
    --lora_target_modules '[q_proj,v_proj]' \
    --train_on_inputs \
    --group_by_length
```

### Inference (`demo`)

[Bloom-Chat demo](https://colab.research.google.com/drive/1MWQsvbanEwt6z4BLFq_BkYdM1WC6pEwA?usp=sharing)

[Bloom-Doctor demo](https://colab.research.google.com/drive/1kiqlFQToWO40L4lGqM8i4UQsHZIJzaPt?usp=sharing)

