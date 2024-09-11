# simple-image-recaptioning

Recaption large (Web)Datasets with `vllm` and save the artifacts. It is NOT a library. It, instead, provides reference points that you're free to use and modify. 

I use the code of this repository for my projects and I don't claim this project to be out of the world. If you want to contribute an enhancement feature, you're more than welcome to open a PR. I'd greatly appreciate it. 

## Getting started

Install the requirements: `pip install -r requirements.txt`. Then run:

```bash
python main.py \
    --data_path="https://huggingface.co/datasets/pixparse/cc3m-wds/resolve/main/cc3m-train-0000.tar"
```

This will recaption a single shard of the CC3M dataset and will serialize the artifacts inside a directory called `sample_outputs`. This directory will have:

* The original image with its hash as its filename.
* A JSON file with the same hash as the filename containing the original and predicted captions.

If you want to use multiple shards then do:

```bash
# full CC3M training set
python main.py \
    --data_path="pipe:curl -s -f -L https://huggingface.co/datasets/pixparse/cc3m-wds/resolve/main/cc3m-train-{0000..0575}.tar"
```

You can allow watermark detection by passing `--detect_watermarks`. Note that this will require the following things:

* `onnx` and `onnxruntime` dependencies.
* Install `pip install git+https://github.com/sayakpaul/watermark-detection`. Then follow [the steps](https://github.com/sayakpaul/watermark-detection?tab=readme-ov-file#onnx-usage-limited-to-convnext-tiny) to obtain the ONNX model needed for watermark detection.

By default, the script will use all the available GPUs. Refer to the `main.py` script for a full list of the supported CLI arguments.

I tested the above commands on two A100s.

## Principles

1. Recaptioning large image datasets has become a da-facto standard for the image generation community. So, I wanted to have a simple-yet-performant utility that would allow me to recaption large image datasets like [CC3M](https://huggingface.co/datasets/pixparse/cc3m-wds). This is why, I chose `vllm` as it provides optimized inference across multiple GPUs off-the-shelf.

2. [`webdataset`](https://github.com/webdataset/webdataset) is a common format used by practitioner to conduct training on large-scale datasets. So, I chose that as an entrypoint. Specifically, I assume that your image-caption pair dataset is already sharded into multiple `webdataset` archives. Refer [here](https://huggingface.co/datasets/pixparse/cc3m-wds) as an example. 

3. I need to be able to use multiple GPUs, overlapping communication and computation.

4. There has to be artifacts serialization. This project serializes the original image, original caption, and the predicted caption in separate threads, not blocking the GPU(s).

5. There has to be watermark detection in the data curation pipeline at minimum. Otherwise, it messes up with the generation quality. This happens _during_ dataloading. To not clog the processes, we make use of ONNX fast CPU-based inferencing.

## Code organization and modification

Ultimately, you'd want to modify the codebase to suit your needs. Below, I provide some pointers.

```bash
.
├── config.py -- specifies the prompt to be used to generate the captions and model id.
├── data_processing.py -- webdataset loading and processing code.
├── main.py -- main entrypoint.
├── model.py -- loads the vllm engine and houses the simple inference function.
└── utils.py -- misc utilities.
```