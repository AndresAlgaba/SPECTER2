# SPECTER 2.0

## Overview
SPECTER 2.0 is a document embedding model for scientific tasks and documents. It builds on the original [SPECTER](https://github.com/allenai/specter) and [SciRepEval](https://github.com/allenai/scirepeval) works, and can be used to generate effective embeddings for multiple task formats i.e Classification, Regression, Retrieval and Search. 

## Usage
We train a base model from scratch on citation links like SPECTER, but our training data consists of 6M (10x) triplets spanning 21 [fields of studies](https://api.semanticscholar.org/CorpusID:256194545). 
Then we train task format specific adapters with SciRepEval to generate multiple embeddings for the same paper.
We represent the input paper as a concatenation of its title and abstract.
For Search type tasks where the input query is a short text rather a paper, use the adhoc query model below to encode it and the retrieval model to encode the candidate papers.  
All the models are publicly available on HuggingFace and AWS S3.

### HuggingFace
|Model|Type|Name and HF link|
|--|--|--|
|Base|Transformer|[allenai/specter_plus_plus](https://huggingface.co/allenai/specter_plus_plus)|
|Classification|Adapter|[allenai/spp_classification](https://huggingface.co/allenai/spp_classification)|
|Regression|Adapter|[allenai/spp_regression](https://huggingface.co/allenai/spp_regression)|
|Retrieval|Adapter|[allenai/spp_proximity](https://huggingface.co/allenai/spp_proximity)|
|Adhoc Query|Adapter|[allenai/spp_adhoc_query](https://huggingface.co/allenai/spp_adhoc_query)|

```python
from transformers import AutoTokenizer, AutoModel

# load model and tokenizer
tokenizer = AutoTokenizer.from_pretrained('allenai/specter_plus_plus')

#load base model
model = AutoModel.from_pretrained('allenai/specter_plus_plus')

#load the adapter(s) as per the required task, provide an identifier for the adapter in load_as argument and activate it
model.load_adapter("allenai/spp_adhoc_query", source="hf", load_as="adhoc_query", set_active=True)

papers = [{'title': 'BERT', 'abstract': 'We introduce a new language representation model called BERT'},
          {'title': 'Attention is all you need', 'abstract': ' The dominant sequence transduction models are based on complex recurrent or convolutional neural networks'}]

# concatenate title and abstract
text_batch = [d['title'] + tokenizer.sep_token + (d.get('abstract') or '') for d in papers]
# preprocess the input
inputs = self.tokenizer(text_batch, padding=True, truncation=True,
                                   return_tensors="pt", return_token_type_ids=False, max_length=512)
output = model(**inputs)
# take the first token in the batch as the embedding
embeddings = output.last_hidden_state[:, 0, :]
```

### AWS S3 via CLI
```bash
mkdir -p specter2_0/models
cd specter2_0/models
aws s3 --no-sign-request cp s3://ai2-s2-research-public/specter2_0/specter2_0.tar.gz .
tar -xvf specter2_0.tar.gz
```
The above commands will copy all the model weights from S3 as a tar archive and extract two folders-base and adapters.

```python
from transformers import AutoTokenizer, AutoModel

# load model and tokenizer
tokenizer = AutoTokenizer.from_pretrained('specter2_0/models/base')

#load base model
model = AutoModel.from_pretrained('specter2_0/models/base')

#load the adapter(s) as per the required task, provide an identifier for the adapter in load_as argument and activate it
model.load_adapter("specter2_0/models/adapters/adhoc_query", load_as="adhoc_query", set_active=True) 
#other possibilities: .../adapters/<classification|regression|proximity>

papers = [{'title': 'BERT', 'abstract': 'We introduce a new language representation model called BERT'},
          {'title': 'Attention is all you need', 'abstract': ' The dominant sequence transduction models are based on complex recurrent or convolutional neural networks'}]

# concatenate title and abstract
text_batch = [d['title'] + tokenizer.sep_token + (d.get('abstract') or '') for d in papers]
# preprocess the input
inputs = self.tokenizer(text_batch, padding=True, truncation=True,
                                   return_tensors="pt", return_token_type_ids=False, max_length=512)
output = model(**inputs)
# take the first token in the batch as the embedding
embeddings = output.last_hidden_state[:, 0, :]
```

### Batch Processing (requires GPU)
To generate the embeddings for an input batch, follow [INFERENCE.md](https://github.com/allenai/scirepeval/blob/main/evaluation/INFERENCE.md).
Create the Model instance as follows:
```python

adapters_dict = {"[CLF]": "allenai/spp_classification", "[QRY]": "allenai/spp_adhoc_query", "[RGN]": "allenai/spp_regression", "[PRX]": "allenai/spp_proximity"}
model = Model(variant="adapters", base_checkpoint="allenai/specter_plus_plus", adapters_load_from=adapters_dict, all_tasks=["[CLF]", "[QRY]", "[RGN]", "[PRX]"])
```
Follow Step 2 onwards in the provided ReadMe.

## Link to training / test datasets

The training and validation triplets have been added to the SciRepEval benchmark, and is available [here](https://huggingface.co/datasets/allenai/scirepeval/viewer/cite_prediction_new/evaluation).
The training data consists of triplets from [SciNCL](https://github.com/malteos/scincl) as a subset.
The model is trained in two stages using [SciRepEval](https://github.com/allenai/scirepeval/blob/main/training/TRAINING.md):
- Base Model: First a base model is trained on the above citation triplets.
``` batch size = 1024, max input length = 512, learning rate = 2e-5, epochs = 2```
- Adapters: Thereafter, task format specific adapters are trained on the SciRepEval training tasks, where 600K triplets are sampled from above and added to the training data as well.
``` batch size = 256, max input length = 512, learning rate = 1e-4, epochs = 6```


## Link to evaluation information

[probably a link to a jupyter notebook, maybe even defined in this repo]

## Verifying and Publishing This Model

```
tt verify -c specter2_0/timo/config.yaml
```

```
tt publish -c specter2_0/timo/config.yaml
```

See [TIMO User Guide](https://github.com/allenai/timo/blob/main/docs/timo-tools/userguide.md) for
more details.


