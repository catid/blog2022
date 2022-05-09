+++
title = "Downsample-then-Upsample"
date = "2021-04-02"
cover = "posts/downsample/flow.jpg"
description = "I tried out a modern approach to image resampling: an end-to-end learned downsampler-then-upsampler."
+++

## Experiments

I tried out a modern approach to image resampling: an end-to-end learned downsampler-then-upsampler.  The application would be to downsample the image, compress it using a traditional image/video compression method, and then upsample the image back to its original size.  This might be useful as part of a video pipeline.

This project ( code = [https://github.com/sunwj/CAR](https://github.com/sunwj/CAR) paper = [Learned Image Downscaling for Upscaling using Content Adaptive
Resampler](https://arxiv.org/pdf/1907.12904.pdf) ) was found by looking at [paperswithcode.com](https://paperswithcode.com/sota/image-super-resolution-on-set5-4x-upscaling), where it presently ranks as the best method by far.

Here are some other example results you can zoom into:

![Example2](example2.jpg)

![Example3](example3.jpg)

When there are single-pixel features, it does struggle as you'd expect.  But it does a good job preserving features that are ~2 pixels or larger.

## Windows Setup

To get this going on a Windows PC with an Nvidia graphics card, I installed:
* [Rapid Environment Editor](https://www.rapidee.com/) - Used to easily edit the PATH variable.
* [Git Bash](https://gitforwindows.org/)
* [CUDA](https://developer.nvidia.com/cuda-downloads)
* [Latest Python 3](https://www.python.org/downloads/)
* [Ninja](https://github.com/ninja-build/ninja/releases)

Make sure CUDA is in the Windows Path.  For me I added: `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.2\bin` and `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.2\libnvvp`.

Make sure Python3 is in the Windows Path.  For me I added: `C:\Users\leon\AppData\Local\Programs\Python\Python39` and `C:\Users\leon\AppData\Local\Programs\Python\Python39\Scripts`.

To install Ninja, I dropped the latest ninja.exe release file into the Python39 folder.

Then open up Git Bash and clone the repo and set up dependencies:

```
git clone git@github.com:sunwj/CAR.git
cd CAR/adaptive_gridsampler

pip install torch==1.8.1+cu111 torchvision==0.9.1+cu111 torchaudio===0.8.1 -f

pip install tqdm
pip install scipy
pip install numpy
pip install matplotlib
pip install scikit-image

python setup.py build_ext --inplace

cd ..
```

I then downloaded the pre-trained models from the link at [https://github.com/sunwj/CAR](https://github.com/sunwj/CAR) and placed the files under a subfolder.

Now it's ready to test!  The scripts by default support only .PNG losslessly compressed images.

```
python run.py --scale 4 --img_dir catid_images --model_dir models --output_dir output
```
