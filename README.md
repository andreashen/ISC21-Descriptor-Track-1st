# Facebook AI Image Similarity Challenge

## setup

### OS
Ubuntu 18.04

### CUDA Version
11.1

### environment

Run this for python env

```
pip install -r requirements.txt
```

### data download

```
mkdir -p input/{query,reference,train}_images
aws s3 cp s3://drivendata-competition-fb-isc-data/all/query_images/ input/query_images/ --recursive --no-sign-request
aws s3 cp s3://drivendata-competition-fb-isc-data/all/reference_images/ input/reference_images/ --recursive --no-sign-request
aws s3 cp s3://drivendata-competition-fb-isc-data/all/train_images/ input/train_images/ --recursive --no-sign-request
```

## train

Run below lines step by step.

```
cd exp

CUDA_VISIBLE_DEVICES=0,1,2,3 python v83.py \
  -a tf_efficientnetv2_m_in21ft1k --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size 1 --rank 0 --seed 9 \
  --epochs 5 --lr 0.1 --wd 1e-6 --batch-size 128 --ncrops 2 \
  --gem-p 1.0 --pos-margin 0.0 --neg-margin 1.0 \
  --input-size 256 --sample-size 1000000 --memory-size 20000 \
  ../input/training_images/
CUDA_VISIBLE_DEVICES=0,1,2,3 python v83.py \
  -a tf_efficientnetv2_m_in21ft1k --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size 1 --rank 0 --seed 90 \
  --epochs 10 --lr 0.1 --wd 1e-6 --batch-size 128 --ncrops 2 \
  --gem-p 1.0 --pos-margin 0.0 --neg-margin 1.0 \
  --input-size 256 --sample-size 1000000 --memory-size 20000 \
  --resume ./v83/train/checkpoint_0004.pth.tar \
  ../input/training_images/

CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 python v86.py \
  -a tf_efficientnetv2_m_in21ft1k --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size 1 --rank 0 --seed 99 \
  --epochs 7 --lr 0.1 --wd 1e-6 --batch-size 128 --ncrops 2 \
  --gem-p 1.0 --pos-margin 0.0 --neg-margin 1.0 \
  --input-size 384 --sample-size 1000000 --memory-size 20000 --weight ./v83/train/checkpoint_0005.pth.tar \
  ../input/training_images/

python v98.py \
  -a tf_efficientnetv2_m_in21ft1k --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size 1 --rank 0 --seed 999 \
  --epochs 3 --lr 0.1 --wd 1e-6 --batch-size 64 --ncrops 2 \
  --gem-p 1.0 --pos-margin 0.0 --neg-margin 1.0 --weight ./v86/train/checkpoint_0005.pth.tar \
  --input-size 512 --sample-size 1000000 --memory-size 20000 \
  ../input/training_images/

python v106.py \
  -a tf_efficientnetv2_m_in21ft1k --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size 1 --rank 0 --seed 99999 \
  --epochs 10 --lr 0.4 --wd 1e-6 --batch-size 16 --ncrops 2 \
  --gem-p 1.0 --pos-margin 0.0 --neg-margin 1.1 --weight ./v98/train/checkpoint_0001.pth.tar \
  --input-size 512 --sample-size 1000000 --memory-size 1000 \
  ../input/training_images/
```

## inference

Note that faiss doesn't work with A100, so I used 4x GTX 1080 Ti for post-process.

```
cd exp

python v106.py -a tf_efficientnetv2_m_in21ft1k --batch-size 128 --mode extract --gem-eval-p 1.0 --weight ./v106/train/checkpoint_0009.pth.tar --input-size 512 --target-set qrt ../input/

# this script generates final prediction result files
python ../scripts/postprocess.py
```

Submission files are outputted here:

- `exp/v106/extract/v106_iso.h5`  # descriptor track
- `exp/v106/extract/v106_iso.csv`  # matching track

descriptor track local evaluation score:

```
{
  "average_precision": 0.929902992274655,
  "recall_p90": 0.8990182328190743
}
```

matching track local evaluation score:

```
{
  "average_precision": 0.929902992274655,
  "recall_p90": 0.8990182328190743
}
```
