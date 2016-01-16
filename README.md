# About

This project uses GANs ([generative adversarial networks](http://papers.nips.cc/paper/5423-generative-adversarial-nets)) to add color to black and white images.
For each such image, the generator network (G) receives its black and white version and outputs a full RGB version of the image (i.e. the black and white image with color added).
That RGB version is then rated (in regards to its quality) by the discriminator (D).
The quality measure is backpropagated through D and then through G.
Thereby G can learn to correctly colorize images.
The architectures used are modifications of the [DCGAN](http://arxiv.org/abs/1511.06434) schema.
See this blog post for an alternative version which uses standard convnets (i.e. no GANs).

Key results:
* If a dataset of images can be generated by a GAN, then a GAN can also learn to add colors to it.
* The task of adding colors seems to be a bit easier than the full generation of images.
* G did not learn to add colors to rather rare and small elements (e.g. when coloring images of christmas trees it didn't add color to presents below the trees, small baubles or clothes of people in the image). This might partly be a limitation of the architecture which uses pooling layers in G (hence small elements might get lost).
* G did not learn to correctly add colors to datasets with high variance (heterogeneous collections of images). It would resort to mostly just adding one or two colors everywhere.
* I experimented with using VGG features but didn't have much success with those. G didn't seem to learn more than without VGG features. My tests were limited though due to hardware constraints (VGG + G + D = three big networks in memory).
* Producing UV values in G and combining them with Y to an YUV image (which is then fed into D) failed. G just wouldn't learn anything. G had to output full RGB images.

# Images

Colorizers were trained on multiple image datasets which were reused from previous projects. (I.e. *multiple* GANs were trained, not just one for all images. That's due to GANs not being very good at handling heterogeneous datasets.)
Besides of the datasets shown below, the MSCOCO 2014 validation dataset was also used, but G failed to learn much on that one (it added mostly just 1-3 uniform colors per image), hence the results of that run are not shown.

Notes:
* There were no splits into training and validation sets (partly due to laziness, partly because GANs in my experience basically never just memorize the training set).
* Training times were usually quite fast (<=2 hours per dataset).
* All generated color images were a little bit blurry, probably because G generated full RGB images instead of just adding color (UV in YUV). As such, it has to learn to copy the Y channel information correctly while still adding colors.

## Human faces

This dataset worked fairly well. Notice the image in the 10th row at the far right. G assigns a skin color to the microphone. Also notice how G usually doesn't color the lips in red.

![Human faces, training set and images colored by G](images/human_faces.jpg?raw=true "Human faces, training set and images colored by G")

*For each tuple: (left) Original image in black and white, (middle) original image in color, (right) Color added by G.*

## Cat faces

This dataset worked fairly well.

![Cat faces, training set and images colored by G](images/cat_faces.jpg?raw=true "Cat faces, training set and images colored by G")

*For each tuple: (left) Original image in black and white, (middle) original image in color, (right) Color added by G.*

## Skies

Here G created sometimes weird mixtures blue and orange. They were not visible on earlier epochs, but those then had weird vertical stripes around the borders of the images.

![Skies, training set and images colored by G](images/skies.jpg?raw=true "Skies, training set and images colored by G")

*For each tuple: (left) Original image in black and white, (middle) original image in color, (right) Color added by G.*

## Baubles

This dataset already caused problems when I tried to generate it (i.e. full image generation, not just colorization). It didn't work too well here either. Baubles often remained colorless. I had to carefully select the optimal epoch to generate half decent images. There are blue blobs in some of the images. These blobs become bigger if the experiment is run longer.

![Baubles, training set and images colored by G](images/baubles.jpg?raw=true "Baubles, training set and images colored by G")

*For each tuple: (left) Original image in black and white, (middle) original image in color, (right) Color added by G.*

## Snowy landscapes

Here G only had to either keep the black and white image or add some blue color, fairly easy task. It mostly learned that and sometimes exaggerated the blue (e.g. by adding it to trees).

![Snowy landscapes, training set and images colored by G](images/snowy_landscapes.jpg?raw=true "Snowy landscapes, training set and images colored by G")

*For each tuple: (left) Original image in black and white, (middle) original image in color, (right) Color added by G.*

## Christmas trees

This dataset worked fairly well. When zooming in you can see that G doesn't color presents, baubles and people's clothings. (E.g. for baubles look at row=9, col=3, for clothes row=8, col=1 and for presents row=1, col=2.)

![Christmas trees, training set and images colored by G](images/christmas_trees.jpg?raw=true "Christmas trees, training set and images colored by G")

*For each tuple: (left) Original image in black and white, (middle) original image in color, (right) Color added by G.*


# Architecture

The architecture of D was a standard convolutional neural net with one small fully connected layer at the end, mostly ReLU activations and some spatial dropout.

G is an upsampling generator, similar to what is described in the DCGAN paper. Before the upsampling part it has an image analyzation part, similar to a standard convolutional layer. The analyzation part takes the black and white image and tries to make sense of it with some convolution and poolings. The black and white image is fed back in at the end so that the analyzation and upsampling parts can focus on the color and don't have to transfer the Y channel information through the layers. The noise layer is fed into the network both at the start and the end for technical simplicity (it just gets joined with the black and white image).

![Human faces, training set and images colored by G](images/human_faces.jpg?raw=true "Human faces, training set and images colored by G")

*Architecture of G. The formulation "Conv K, 3x3, relu, BN" denotes a convolutional layer with K kernels/planes, each with filter size 3x3, ReLU activation and batch normalization. Max pooling is over a 2x2 area. Upsampling layers increase height and width each by 2. The noise layer and the black and white image usually both have size 1x64x64.*

D had about 3 million parameters (exact value depends on the input image size), G about 0.8 million.

# Usage

Requirements are:
* Torch
  * Required packages (most of them should be part of the default torch install, install missing ones with `luarocks install packageName`): `cudnn`, `nn`, `pl`, `paths`, `image`, `optim`, `cutorch`, `cunn`, `cudnn`, `dpnn`, `display`
* Image datasets have to be downloaded from previous projects and will likely require Python 2.7. You can however use your own dataset, provided that the images in that one are square or have an aspect ratio of 1.5 (e.g. height=48, width=32). Other ratios might work but haven't been tested.
  * [Human faces](https://github.com/aleju/face-generator)
  * [Cat faces](https://github.com/aleju/cat-generator)
  * [Skies](https://github.com/aleju/sky-generator)
  * [Christmas trees, baubles, snowy landscapes](https://github.com/aleju/christmas-generator)
* NVIDIA GPU with cudnn3 and 4GB or more memory

To train a network:
* `~/.display/run.js &` - This will start `display`, which is used to plot results in the browser
* Open http://localhost:8000/ in your browser (`display` interface)
* Open a console in the repository directory and then `th train_rgb.lua --dataset="DATASET_PATH" --height=64 --width=64`, where `DATASET_PATH` is the filepath to the directory containing all your images (must be jpg). `height` and `width` resemble the size of the *generated* images. Your source images in that directory may be larger (e.g. 256x256). Only 32x32 (height x width), 64x64 and 32x48 were tested. Other values might result in errors. Note: Training keeps running until stopped manually with ctrl+c.

To continue a training session use `th train_rgb.lua --dataset="DATSET_PATH" --height=64 --width=64 --network="logs/adversarial.net"`.
To sample images (i.e. colorize images from the training set) use `th sample.lua` (should automatically reuse dataset directory, height and width).
