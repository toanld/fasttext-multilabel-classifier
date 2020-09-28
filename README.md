# Multilabel Text Classifier with fastText

Created by [Fascebook AI Research](https://research.fb.com/category/facebook-ai-research/), [fastText](https://fasttext.cc/) is a library for efficient learning of words and classification of texts:
> FastText is an open-source, free, lightweight library that allows users to learn text representations and text classifiers. It works on standard, generic hardware. Models can later be reduced in size to even fit on mobile devices.

This project applies fastText to perform multilabel text classification and dockerizes the trainer and classifier for easy deployment.

## Usage

The instructions of building and running the classification trainer and server are described as follows. You may build the docker images from the source or [pull the docker images directly from the Docker Hub](#pull-docker-images-from-docker-hub).

### Prepare the dataset as a sqlite database
The training data is expected to be given as a [sqlite](https://www.sqlite.org/index.html) database. It consists of two tables, `texts` and `labels`, storing the texts and their associated labels:
```SQL
CREATE TABLE IF NOT EXISTS texts (
    id TEXT NOT NULL PRIMARY KEY,
    text TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS labels (
    label TEXT NOT NULL,
    text_id text NOT NULL,
    FOREIGN KEY (text_id) REFERENCES texts(id)
);
CREATE INDEX IF NOT EXISTS label_index ON labels (label);
CREATE INDEX IF NOT EXISTS text_id_index ON labels (text_id);
```

An empty example sqlite file is in [`example/train.db`](https://github.com/yam-ai/fasttext-multilabel-classifier/blob/master/example/train.db).

Let us take the [toxic comment dataset](https://www.kaggle.com/c/jigsaw-toxic-comment-classification-challenge/data) published on [kaggle](https://www.kaggle.com/) as an example. (Note: you will need to create a kaggle account in order to download the dataset.) The training data file `train.csv` (not provided by this repository) in the downloaded dataset has the following columns: `id`, `comment_text`, `toxic`, `severe_toxic`, `obscene`, `threat`, `insult`, `identity_hate`. The last six columns represent the labels of the `comment_text`.

The python script in [`example/csv2sqlite.py`](https://github.com/yam-ai/fasttext-multilabel-classifier/blob/master/example/csv2sqlite.py) can process `train.csv` and save the data in a sqlite file `train.db`.

To convert `train.csv` to `train.db`, run the following commands:
```sh
python3 csv2sqlite.py -i /downloads/toxic-comment/train.csv -o /repos/bert-multilabel-classifier/example/train.db
```
You can also use the `-n` flag to convert only a subset of examples in the training csv file to reduce the training database size. For example, you can use `-n 1000` to convert only the first 1,000 examples in the csv file into the training database. This may be necessary if there is not enough memory to train the model with the entire raw training set or you want to shorten the training time.

### Tune the parameters
The training and serving parameters can be modified in [`settings.py`](https://github.com/yam-ai/fasttext-multilabel-classifier/blob/master/settings.py).

### Train the model
Build the docker image for training:
```sh
docker build -f train.Dockerfile -t classifier-train .
```  

Run the training container by mounting the above volumes:
```sh
docker run -v $TRAIN_DIR:/train -v $MODEL_DIR:/model classifier-train
```

* `TRAIN_DIR` is the full path of the input directory that contains the sqlite DB `train.db` storing the training set, e.g., `TRAIN_DIR=/data/example/train/`.
* `MODEL_DIR` is the full path to the output directory that stores the fastText trained model `model.bin` to be generated, e.g., `MODEL_DIR=/data/example/model/`.

If you want to override the default settings with your modified settings, for example, in `/data/example/settings.py`, you can add the flag `-v /data/example/settings.py:/srv/settings.py`.

### Serve the model
Build the docker image for the classifier server:
```sh
docker build -f serve.Dockerfile -t classifier-serve .
```

Run the serving container by mounting the trained model file and exposing the port:
```sh
docker run -v $MODEL_DIR:/model -p 5001:5001 classifier-serve
```

* `MODEL_DIR` is the full path of the directory that stores the trained model `model.bin` generated in the above step.

If you want to override the default settings with your modified settings, for example, in `/data/example/settings.py`, you can add the flag `-v /data/example/settings.py:/srv/settings.py`.

### Post an inference HTTP request

Make an HTTP POST request to `http://localhost:5001/classifier` with a JSON body which contains the texts to be labeled, like the following (two Albert Einstein quotes):
```json
{ 
   "texts":[ 
      { 
         "id":0,
         "text":"Three great forces rule the world: stupidity, fear and greed."
      },
      { 
         "id":1,
         "text":"Put your hand on a hot stove for a minute, and it seems like an hour. Sit with a pretty girl for an hour, and it seems like a minute. That's relativity."
      }
   ]
}
```

The classifier returns a list of scores for the labels, indicating the likelihoods of the labels assigned to the input texts:
```json
[ 
   { 
      "id":0,
      "scores":{ 
         "toxic":1.0000100135803223,
         "insult":0.148057222366333,
         "obscene":0.0023331623524427414,
         "identity_hate":0.0007654056535102427,
         "threat":1.0000003385357559e-05,
         "severe_toxic":1.0000003385357559e-05
      }
   },
   { 
      "id":1,
      "scores":{ 
         "toxic":0.9919480085372925,
         "insult":0.4225146174430847,
         "obscene":0.3998216390609741,
         "identity_hate":1.0000003385357559e-05,
         "threat":1.0000003385357559e-05,
         "severe_toxic":1.0000003385357559e-05
      }
   }
]
```

You can test the classifier API using `curl` as follows:
```sh
curl -X POST http://localhost:5001/classifier -H "Content-Type: application/json" -d $'{"texts":[{"id":0,"text":"Three great forces rule the world: stupidity, fear and greed."},{"id":1,"text":"Put your hand on a hot stove for a minute, and it seems like an hour. Sit with a pretty girl for an hour, and it seems like a minute. That\'s relativity."}]}'
```

### Pull docker images from Docker Hub
We have published the docker images on the [Docker Hub](https://hub.docker.com/r/yamai/fasttext-multilabel-classifier) so that you need not build the docker images from the source. You can pull them directly from the Docker Hub as follows:

```sh
docker pull yamai/fasttext-multilabel-classifier:train-latest
docker pull yamai/fasttext-multilabel-classifier:serve-latest
```

After these images are successfully pulled, you can run the training or serving container as follows:

```sh
docker run -v $TRAIN_DIR:/train -v $MODEL_DIR:/model yamai/fasttext-multilabel-classifier:train-latest
```
or
```sh
docker run -v $MODEL_DIR:/model -p 5001:5001 yamai/fasttext-multilabel-classifier:serve-latest
```

## Profesional services

If you need any supporting resources or consultancy services from [YAM AI Machinery](https://www.yam.ai), please find us at:
* https://www.yam.ai
* https://twitter.com/theYAMai
* https://www.linkedin.com/company/yamai
* https://www.facebook.com/theYAMai
* https://github.com/yam-ai
* https://hub.docker.com/u/yamai
