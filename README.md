# Semi-Automatic NLI Data Collection

This is a repository for data and code accompanying paper "Asking Crowdworkers to Write Entailment Examples: The Best of Bad Options".

## Datasets

The five datasets described in the paper are available under `data/` directory: `base_news`, `base_wiki`, `sim_news`, `sim_wiki`, and `translate_wiki`. Each of the dataset comes with a training set and a test set, both in `.jsonl` format. Please refer to the paper for the statistics for each of dataset.

## License

We use premises taken from the [English Gigaword Fifth Edition](https://catalog.ldc.upenn.edu/LDC2011T07), English Wikipedia and Simple Wikipedia (downloaded May 2020), and [WikiMatrix](https://github.com/facebookresearch/LASER/tree/master/tasks/WikiMatrix). The English Gigaword is distributed under the [LDC User Agreement license](https://catalog.ldc.upenn.edu/LDC2011T07). Wikipedia is licensed under Creative Commons Attribution-ShareAlike 3.0 Unported License (CC-BY-SA) and the GNU Free Documentation License (GFDL). 


## Data Preprocessing

### Requirements

- spacy
- beautifulsoup4
- GNU Parallel

### Usage

Code for preprocessing Gigaword and Wikipedia data (adapted from [here](https://github.com/nelson-liu/flatten_gigaword)) can be found under `script/preprocess`. 

For Gigaword, you need to unzip the data. You can then run the `parse_gigaword.sh` which has three arguments as inputs: (1) the path of unzipped Gigaword news source, (2) the output directory for parsed file, and (3) the number of files to be processed at once. For example, to preprocess `afp_eng`, you can use the following command:
```
bash parse_gigaword.sh ./data/gigaword_eng_5/afp_eng ./data/gigaword_preprocessed 5
```
The output directory for each news source will have three sub-directories:
- `text`: consisting segmented sentences, each with a unique id. Each id is a concatenation of the sentence's article id, paragraph id, and the position of the sentence in that paragraph.
- `headlines`: a pickle file that stores headlines for each article id.
- `types`: a pickle file that stores the document type for each article id.

For Wikipedia, first download the dump file, and then extract them using [WikiExtractor](http://medialab.di.unipi.it/wiki/Wikipedia_Extractor):
```
python wikiextractor/WikiExtractor.py -b -l 1M -o extracted data/enwiki-latest-pages-articles.xml.bz2
```
You can then parse the Wikipedia articles using the following command:
```
bash scripts/parse_enwiki.sh ./data/extracted_data ./data/wikipedia_preprocessed 5
```
The output directory will consist of three sub-directories:
- `first_par`: first paragraph sentences of each article (using as seed sentences to train index clusters).
- `text`: sentences with a unique id. Each id is a concatenation of article id, paragraph id, and sentence position in the paragraph.
- `metadata`: metadata for each sentence, consisting of Wikipedia document title, and hyperlinks in the sentence. These information are used for reranking.


## Indexing and Retrieval

### Requirement

- [FAISS](https://github.com/facebookresearch/faiss) library, we use faiss-gpu 1.6.0
- fasttext >= 0.9.1
- spaCy

### Gigaword

The script for indexing and retrieval can be found at `scripts/indexing_retrieval/index_retrieve.py`.
To build an index for a news source (e.g., `wpb_eng`), you can run the following command:
```
python scripts/index_retrieve.py \
				--data-dir=./data/gigaword_preprocessed \
				--embedding-path=./data/cc.en.300.bin \
				--seed-file=./seeds/txt_wpb_eng.txt \
				--mode="index" \
				--source-name=wpb_eng \
				--out-dir=./index
```
After running the command, you can find the built index inside the `--out-dir` directory. There will be three files `.index`, `.vocab`, and `.ts` which are needed during retrieval.

For retrieval, you would first need a set of sentences as queries. We give an example in `scripts/indexing_retrieval/sample_queries.txt`. You can also use the provided `generate_queries.py` to generate your own queries. After that, you can retrieve similar sentences for each query using the following command:
```
python scripts/index_retrieve.py \
				--embedding-path=./data/cc.en.300.bin \
				--mode="retrieve" \
				--index-prefix=./index/wpb_eng_100 \
				--queries=sample_queries.txt \
				--output-path=./outputs/text/wpb_eng.out \
				--source-name=wpb_eng
```
The retrieval output can then be found at the location of `--output-path` path. Each set of retrieved sentence is separated by a new line. For each set, the first line would be the query, which will be followed by the retrieved sentences, sorted by the FAISS similarity score (first column, sorted by the FAISS similarity score from the most similar to the less similar ones).


### Wikipedia

Indexing and retrieval for Wikipedia is similar to Gigaword, except that we only build one index and use the first paragraph of each article as seed sentences to train the index clusters.

To build an index, you can run the following command:
```
python scripts/index_retrieve_enwiki.py \
				--data-dir=./data/wikipedia_preprocessed \
				--embedding-path=./data/cc.en.300.bin \
				--mode=index \
				--out-dir=./index
```
and for retrieval you can run the following:
```
python scripts/index_retrieve_enwiki.py \
				--index-prefix=./index/enwiki \
				--output-path=outputs/test.out \
				--queries=queries.txt \
				--embedding-path=./data/cc.en.300.bin \
				--mode=retrieve
```

## Translating WikiMatrix

We used and followed the default setup from [Opus-MT](https://github.com/Helsinki-NLP/Opus-MT). 

## Reranking

### Requirements

- jiant v1.2
- Optuna 1.0.0

### Usage

First, put all retrieved outputs to a folder, for example `./retrieved_outputs`. Then run the tagger:
```
python scripts/reranking/tagger.py ./retrieved_outputs
```
This will generate `.tagged` files, which are the retrieved output files tagged with their features (subjects, noun phrases, etc.)

After that, you can run the rerank them using `reranker.py`:
```
python scripts/reranking/reranker.py \
				--num_queries=100 \
				--input_path=./retrieved_outputs
				--output_prefix=gigaword_reranked
```
where `--num_queries` is the number of queries to be reranked, `--input_path` is the path of retrieved outputs, and `--output_prefix` is the name of the reranked output. The script will generate two files, one in `.txt` format, and another one in `.jsonl` format.

To rerank Wikipedia output, you can use `reranker_enwiki.py`. There is one additional input `--link_dict` which lists the hyperlinks for each document in the retrieved output. You can generate this file using `extract_links.py` script. We provide examples of each file format in `scripts/reranking` directory.
```
python bopt/reranker_enwiki.py \
				--num_queries 100 \
				--input_path=enwiki.tagged \
				--output_prefix=enwiki \
				--link_dict=enwiki.link
```


## Citation

```
@inproceedings{vania2020asking,
    title = "{Asking Crowdworkers to Write Entailment Examples: The Best of Bad Options}",
    author = "Vania, Clara  and
      Chen, Ruijie  and
      Bowman, Samuel R.",
    booktitle = "Proceedings of the 1st Conference of the Asia-Pacific Chapter of the Association for Computational Linguistics and the 10th International Joint Conference on Natural Language Processing",
    month = dec,
    year = "2020",
    address = "Online",
    publisher = "Association for Computational Linguistics"
}
```

