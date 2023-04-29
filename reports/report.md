# Inspecting power systems

## **Introduction**

The ability to identify key components from drone imagery could be a extremly useful as part of another pipeline to for example detect faults or relating real world components to virtual components. For example in using a drone to map powergrids for [NVE Atlas](https://atlas.nve.no/Html5Viewer/index.html?viewer=nveatlas#).

**Task defined in the project catalog:**

<span style="color:hsla(280, 30%, 50%, 1)">

> Drones enable visual inspection of power lines. Combined with computer vision and autonomous navigation, they can provide a cheap and accurate way to assess conditions and detect defects, even in remote or rugged areas. Drones equipped with high-resolution cameras can capture images of power lines from different angles, allowing fordetailed inspections of the lines for any signs of damage or wear.Equipped with thermal sensors able to see “hot spots,” their utility is further expanded, enabling the detection of loose connections or other equipment failures. The drone data can be transmitted in real-time to a remote monitoring station, allowing for real-time monitoring of the power lines and rapid response to any detected issues Goals
>* Create a system to precisely locate power lines in images collected from drones.
>* To broaden the project’s perspective, you can investigate ways to detect and classify power stations from aerial imagery.

</span>

**My Goal:**

* Detect powerlines, masts, transformers and top-covers from drone images using a convolutional neural network (CNN), combine these to an ensemble and then potentially use that in a pipeline for mapping energy infrastructure with location data to physical components.  

**Open datasets:**

1. PLD-UAV data set: [GitHub](https://github.com/SnorkerHeng/PLD-UAV). This dataset has labeled data of power line images in both urban and mountainus terrain. Limited to only images of powerlines. Not able to implement anything else than power line detection.
2. OGNet data set: [Stanford ML Group](https://stanfordmlgroup.github.io/projects/ognet/). This is a data set of oil refineries. Still, the methods and experience you’ll obtain from studying this data set will be highly generalizable to similar problems (if you get a hold of labeled data).
3. Self collected data using a DJI-mini 2 drone flown in Sogn og Fjordane, Norway. [GitHub link](https://github.com/snadderbyte/powerstations). These images were croped into a 4:3 ratio, and then labeled using label-studio.

I will be using the third dataset for this project.

## **Problem**

<i>

Example cases:

* A region loses power due to a tree falling over the power lines. To find where the failure is the drone is flown along the lines until the failure is found. ![Image of pine tree fallen over powerlines Source: https://www.europower.no/nett/-ryddebeltene-var-brede-nok/1-2-239377](https://images-global.nhst.tech/image/UmZsUzJqUEdHbS93dmJRdW55ZlNyditXdXNLaHQxNERRM255czk3cEpkYz0=/nhst/binary/774fbd074bb3fa4184e40763539c94de?image_version=1200)
* Power lines are critical infrastructure and therefore require regular maintenance. A drone could be flown along the lines to find the irregularities and other artifacts, such as missing or damaged top covers, that could compromise the infrastructure.

</i>

## **Components**



### **The mast**

Masts are the either wooden or metalic structures on which the power system components are mounted. 

### **The powerlines**

The powerlines are lines across each image often in black, white or grey. 

### **Transformer and Top-cover**

Some smaller tranformers are often mounted directly on the masts itself. They are metallic boxes with lots of radiator lines for cooling on the outside. 

Top-covers are mounted on top of wooden masts to keep moisture and rainwater away from the core of the wood. These are often just black plastic cicles.

The picture below show a wooden mast with a transformer and one missing top-cover. Top-cover are the black circles on top of each pole these play a critical role in safety. Without them the masts rots extremly fast, and this rot is extremly hard to detect, because the mast will remain stading even though its rot has hollowed out the core of the mast. Transformers are the metal boxes attached half-way up the mast. ![Mast with transformer and one missing top-covers](assets/mast_with_transformer.jpeg)

## **Data**

The dataset that will be used for this project is made up of images taken in the forests of Northern Vestland, Norway. They were all taken in late march 2023.

<i>

<span style="color:hsla(120, 70%, 50%, 1)">

Upps: 
* New images
* Good spread of infrastructue size
* High quality images

</span>

<span style="color:hsla(0, 80%, 50%, 1)">

Downs:
* Small dataset size
* Low count of transformers and top-covers images
* Large images, power lines can be quite small(pixelsize)
* "Bad" labels

</span>

</i>

Both images and mask are saved as image data. The image under show mask overlayed on the original image.
![Show_batch on mast](assets\show_batch_on_mast.png)
![Show_batch on linje](assets\show_batch_on_linje.png)

## **Models**

The initial approach uses a convolutional neural network (CNN) and a transfer learning approach to do sematic segmentation. Through the fastai API. With the standard ``vison_learner`` from fastai and using a pretrained resnet backbone. The models and transformers are qutie simple.

```python
linje = DataBlock(blocks=(ImageBlock, MaskBlock(codes)), 
    get_items=_get_file_names, 
    splitter= RandomSplitter(valid_pct=0.2),
    get_y = get_msk,
    get_x = get_img,
    # Ratioresize could not work for me. Resizes, but keeps aspectratio
    item_tfms=Resize(348, 224),
    batch_tfms=[*aug_transforms(size=224, min_scale=0.75), 
        Normalize.from_stats(*imagenet_stats, cuda=True)],)

learn = unet_learner(dls, arch=resnet34, pretrained=True)
```

This model uses U-Net architecture with a pretrained resnet backbone. 

![U-Net achitecture with resnet backbone src: https://www.researchgate.net/figure/Generator-network-a-U-net-b-ResNet-c-ResU-net-d-Details-of-blocks_fig1_333238725](https://www.researchgate.net/publication/333238725/figure/fig1/AS:761194839293953@1558494400635/Generator-network-a-U-net-b-ResNet-c-ResU-net-d-Details-of-blocks.jpg)

A ``RandomSplitter`` was used to split the training and validation data. Custom getter where made for the image and mask. The images was first resized down to 348 by 224, then cropped down to 224 by 224. 

Because the model uses a backbone, resnet, which is trained on ImageNet data the data is normalized using ``Normalize.from_stats(*imagenet_stats, cuda=True)``. (without ``cuda=True`` i had issues with gpu and pytorch)

All models where trained for various ammounts of epochs. The power-line model performed generally better the longer it was trained for. The mast model also performed well when trained for 100 epochs, however the validation loss would hit 0.13 ± 0.02. While the training loss lowered. This could be a sign of overfitting. Training beyond 100 epochs would not give any performance gain.

Resnet18 and Resnet34 were used for training, but due to memory issues with the GPU any resnet achitectures with more layers would create memory issues. 
> My GPU has 18GB of memory and the training was using all of it, and partitioning the disk for more. This increased training times to too long

## **Conclusion**

These models are only a proof of concept and overall could not be used in a production system. Top-cover and transformer models are not able to accurately mark their targets. 

The mast model is the best performer overall. Allthough it is most accurate on wooden masts. 

The power-line models performed decent and was able mark some of their target, however it think that branches or other natural lines in the background was powerlines.

I will say I belive I failed my goals. I complete an ensemble

## **Discussion**

Throughout the project I made several mistakes. In no praticular order:
* Labeling data took longer than i expected.
* Low quantity of data.
* Sloppy labeling
* Not using all the tools avaliable. (openCV and such)
* GPU-issues when training
* Other time manageing issues

Something i especially regret is not using contour analysis for creating masks.

### **What i would do differently**

* I would label more data.

* Eventhough mast was the model that performed the best i think i would use a object-detection approach. I would also use this approach for the transformer and the top-cover

* The power-line model I think I would stick with a segmentation approach. However U-Net I would experiment with other architectures. 