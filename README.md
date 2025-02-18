# Self-critical Sequence Training for Image Captioning

This is an unofficial implementation for [Self-critical Sequence Training for Image Captioning](https://arxiv.org/abs/1612.00563). 

The latest topdown and att2in2 model can achieve 1.12 Cider score on Karpathy's test split after self-critical training.

This is based on Ruotian's [self-critical.pytorch](https://github.com/ruotianluo/self-critical.pytorch) repository.

## Requirements
Python 2.7 and PyTorch 0.2 (along with torchvision)

## Pretrained models

You need to download pretrained resnet model <b>resnet101.pth</b> or <b>resnet50.pth</b> for both training and evaluation. The models can be downloaded from [here](https://drive.google.com/open?id=0B7fNdx_jAqhtbVYzOURMdDNHSGM), and should be placed in `data/imagenet_weights` folder.

Pretrained image caption models (on coco dataset) are provided [here](https://drive.google.com/open?id=0B7fNdx_jAqhtdE1JRXpmeGJudTg).

## Train your own network on COCO

### Download COCO dataset and preprocessing

First, download the coco images from [link](http://mscoco.org/dataset/#download). We need 2014 training images and 2014 val. images. You should put them in the `image/train2014/` and `image/val2014/` directories, denoted as `$IMAGE_ROOT`.

Download preprocessed coco captions from [link](http://cs.stanford.edu/people/karpathy/deepimagesent/caption_datasets.zip) from Karpathy's homepage. Extract <b>`dataset_coco.json`</b> from the zip file and copy it in to `data/`. This file provides preprocessed captions and also standard train-val-test splits.

Once we have these, we can now invoke the `prepro_*.py` script, which will read all of this in and create a dataset (two feature folders, a hdf5 label file and a json file).

```bash
$ python scripts/prepro_labels.py --input_json data/dataset_coco.json --output_json data/cocotalk.json 
--output_h5 data/cocotalk
$ python scripts/prepro_feats.py --input_json data/dataset_coco.json --output_dir data/cocotalk 
--images_root image
```

`prepro_labels.py` will map all words that occur <= 5 times to a special `UNK` token, and create a vocabulary for all the remaining words. 

The image information and vocabulary are dumped into <b>`cocotalk.json`</b> and discretized caption data are dumped into <b>`cocotalk_label.h5`</b>. Both are stored in `/data` folder.

`prepro_feats.py` extract the resnet101 features (both fc feature and last conv feature) of each image. The features are saved in <b>`.npy`</b> in <b>`data/cocotalk_fc`</b> and <b>`.npz`</b> in <b>`data/cocotalk_att`</b>, and resulting files are about 200GB.

(Check the prepro scripts for more options, like other resnet models or other attention sizes.)


### Start training

```bash
$ python train.py --id fc --input_json data/cocotalk.json --input_fc_dir data/cocotalk_fc 
--input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 --batch_size 48 
--learning_rate 1e-3 --learning_rate_decay_start 0 --scheduled_sampling_start -1 
--checkpoint_path log_fc --save_checkpoint_every 800 --val_images_use 1000 --max_epochs 300 
--caption_model topdown --seq_per_img 1
```

The train script will dump checkpoints into the folder specified by `--checkpoint_path` (default = `save/`). We only save the best-performing checkpoint on validation and the latest checkpoint to save disk space.

To resume training, you can specify `--start_from` option to be the path saving `infos.pkl` and `model.pth` (usually you could just set `--start_from` and `--checkpoint_path` to be the same).

The current command use scheduled sampling, you can also set scheduled_sampling_start to -1 to turn off scheduled sampling.

If you'd like to evaluate BLEU/METEOR/CIDEr scores during training in addition to validation cross entropy loss, use `--language_eval 1` option, but don't forget to download the [coco-caption code](https://github.com/tylin/coco-caption) into `coco-caption` directory.

For more options, see `opts.py`. 

A few notes on training. To give you an idea, with the default settings one epoch of MS COCO images is about 11000 iterations. After 1 epoch of training results in validation loss ~2.5 and CIDEr score of ~0.68. By iteration 60,000 CIDEr climbs up to about ~0.84 (validation loss at about 2.4 (under scheduled sampling)).

### Train using self-critical

First you should preprocess the dataset and get the cache for calculating cider score:
```
$ python scripts/prepro_ngrams.py --input_json data/dataset_coco.json --dict_json data/cocotalk.json 
--output_pkl data/coco-train --split train
```

And also you need to clone my forked [cider](https://github.com/ruotianluo/cider) repository.

Then, copy the model from the pretrained model using cross entropy. (It's not mandatory to copy the model, just for back-up)
```
$ bash scripts/copy_model.sh fc fc_rl
```

Then
```bash
$ python train.py --id fc_rl --caption_model fc --input_json data/cocotalk.json 
--input_fc_dir data/cocotalk_fc --input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 
--batch_size 10 --learning_rate 1e-3 --start_from log_fc_rl --checkpoint_path log_fc_rl 
--save_checkpoint_every 6000 --language_eval 1 --val_images_use 5000 --self_critical_after 30
```

Starting self-critical training after 30 epochs, the CIDEr score goes up to 1.05 after 600k iterations (including the 30 epochs pertraining).

## Generate image captions

### Evaluate on raw images
Now place all your images of interest into a folder, e.g. `blah`, and run
the eval script:

```bash
$ python eval.py --model log_fc_rl/model-best.pth --infos_path log_fc_rl/infos_fc_rl-best.pkl 
--image_folder image/test2014 --num_images 10
```

```bash
$ python eval.py --model log_fc/model-best.pth --infos_path log_fc/infos_fc-best.pkl 
--image_folder image/test2014 --num_images 10
```

This tells the `eval` script to run up to 10 images from the given folder. If you have a big GPU you can speed up the evaluation by increasing `batch_size`. Use `--num_images -1` to process all images. The eval script will create an `vis.json` file inside the `vis` folder, which can then be visualized with the provided HTML interface:

```bash
$ cd vis
$ python -m SimpleHTTPServer
```

Now visit `localhost:8000` in your browser and you should see your predicted captions.

### Evaluate on Karpathy's test split

```bash
$ python eval.py --dump_images 0 --num_images 5000 --model log_fc_rl/model-best.pth 
--infos_path log_fc_rl/infos_fc_rl-best.pkl --language_eval 1 
```

```bash
$ python eval.py --dump_images 0 --num_images 500 --model log_fc/model-best.pth 
--infos_path log_fc/infos_fc-best.pkl --language_eval 1 
```

The default split to evaluate is test. The default inference method is greedy decoding (`--sample_max 1`), to sample from the posterior, set `--sample_max 0`.

**Beam Search**. Beam search can increase the performance of the search for greedy decoding sequence by ~5%. However, this is a little more expensive. To turn on the beam search, use `--beam_size N`, N should be greater than 1.

## Miscellanea

**Train on other dataset**. It should be trivial to port if you can create a file like `dataset_coco.json` for your own dataset.

