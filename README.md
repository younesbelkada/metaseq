# How to load the small model

1. Clone this repo and install all dependencies:

```
git clone https://github.com/facebookresearch/metaseq.git
cd metaseq
pip3 install -e .
```

2. Install Megatron LM as described in the [official setup.md](https://github.com/facebookresearch/metaseq/blob/main/docs/setup.md).

```
git clone --branch fairseq_v2 https://github.com/ngoyal2707/Megatron-LM.git
cd Megatron-LM
pip3 install six regex
pip3 install -e .
```

3. Create a directory where you save the model and tokenizer
```
mkdir -p add_opt
cd add_opt
```

4. Download the 350m model as shown [here](https://github.com/facebookresearch/metaseq/tree/main/projects/OPT).

```
wget https://dl.fbaipublicfiles.com/opt/v1_20220502/350m/reshard.pt
```

5. Comment [this line](https://github.com/ngoyal2707/Megatron-LM/blob/ae0b844c1f6725c3433a95e42cac760b3885170b/megatron/initialize.py#L65) since the rank is only needed to initialize different random seeds accross pp ranks.

6. Create the following Python script:
```py
import os

from transformers import BartTokenizerFast
from megatron import get_args
from megatron.initialize import initialize_megatron
from metaseq import checkpoint_utils

path = "/home/patrick/add_opt"

tokenizer = BartTokenizerFast.from_pretrained("facebook/bart-large")
tokenizer.save_pretrained(path)

# arguments taken from: https://arxiv.org/pdf/2205.01068.pdf | table 1
initialize_megatron(args_defaults={
    "micro_batch_size": 1, 
    "num_layers": 24, 
    "hidden_size": 1024, 
    "num_attention_heads": 16,
    "max_position_embeddings": 2048, # TODO check if it is the correct args
    "encoder_seq_length": 2048 # TODO check if it is the correct args
})

checkpoint = checkpoint_utils.load_model_ensemble_and_task(
#    [os.path.join(path, "reshard-model_part-0.pt"), os.path.join(path, "reshard-model_part-1.pt")],
    [os.path.join(path, "reshard.pt")],
    arg_overrides={
        "vocab_filename": os.path.join(path, "vocab.json"),
        "merges_filename": os.path.join(path, "merges.txt"),
    }
)

model = checkpoint[0][0].eval()


# forward passes
def single_batch_forward_logits(prompts):
    input_ids = tokenizer(prompts, return_tensors="pt", padding=True, truncation=True).input_ids
    logits = model(input_ids)[0]
    return logits

prompts = [
    "Today is a beautiful day and I want to",
    "In the city of",
    "Paris is the capital of France and",
    "Computers and mobile phones have taken",
]

meta_logits = single_batch_forward_logits(prompts)
import ipdb; ipdb.set_trace()
```

6. Now run:

```python
torchrun run_model.py --pipeline-model-parallel-size 1 --tensor-model-parallel-size 1
```

Make sure the logits of the HF models correspond to the `meta_logits` values.
