### "All the time the sun / Is wheeling out of a dark bright ground."

<p><a href="https://s3.amazonaws.com/rossgoodwin/neuralsnap/145215747053_img1.jpg"><img src="https://s3.amazonaws.com/rossgoodwin/neuralsnap/145215747053_img1.jpg" width="275px" class="img-thumbnail"></a></p><p><span style="color:#D64541;">A close up of a red and white striped umbrella</span> and the streets are still still and the sun is still a little and the sun is still and the sun is still and the sun is still and the sun is gone.</p><p>The sun is singing and the sun is still and the sun is looking for the sun.</p><p>The sun is still there and the sun is still and the sun is down and the sun is still and the sun is gone.</p><p>The sun is still there and the sun is gone.</p><p>The sun is still and the sun is still there.</p><p>The sun is dead.</p><p>The sun is still, and the sun is shining in the wind.</p>
<p><span style="color:#D64541;">A close up of a red and white striped umbrella</span>, and the post office was a fire of stone in the middle of the bar. The streetcar remains of the police. The bar was a little strange and sharp as the man who had the strength to stay away from the problem. The little girl was a big fish with a green shoe which had hardly a shot and the party of the storm was being considered a strange story.</p>
<p><span style="color:#D64541;">A close up of a red and white striped umbrella</span>, and the mountains are lighted and still there is no definition in the morning of a little of morning to the first thing that makes the moon sink under the space that will be gone from the stars.</p><p>I want to see it like this has been so awkward and unwanted. I shall see it all out of the sea of the past. Only the sound of the trees are coming up like a bone to be put on a strange ship.</p>
<p><span style="color:#D64541;">A close up of a red and white striped umbrella</span>, and the living and the dark and the red with the sleep of a hand, and the wind remains the sun and the high leaves.</p><p>The sun is beautiful, and there is no desire to feel the mind off the sea.</p><p>In the middle of the place, the moon is opened with the human form.</p><p>The world is all the streets are all there.</p><p>The land of the broken light of life is a procession</p><p>Of many million men in lonely arms</p><p>On the page of the barn. All the time the sun</p><p>Is wheeling out of a dark bright ground.</p>

*(Painting by Mark Rothko, poetry generated by NeuralSnap)*

# NeuralSnap

A new year, a new word camera: NeuralSnap uses artificial neural networks to generate poetry from images.

By [Ross Goodwin](http://rossgoodwin.com), 2016

## More Output Samples

* [Ikeda installation](https://s3.amazonaws.com/rossgoodwin/neuralsnap/145237613476_Epoch_64_24_Loss_1_1434_Dropout_0_25_512_16_8_9_7_Ikeda.html)
* [Airplanes](https://s3.amazonaws.com/rossgoodwin/neuralsnap/145240870834_Epoch_64_24_Loss_1_1434_Dropout_0_25_512_16_6_7_8_9_Planes.html)
* [Keys](https://s3.amazonaws.com/rossgoodwin/neuralsnap/145240853879_Epoch_64_24_Loss_1_1434_Dropout_0_25_512_16_6_7_8_9_Keys.html)
* [Mies Van Der Rohe](https://s3.amazonaws.com/rossgoodwin/neuralsnap/145232647105_Epoch_83_51_Loss_1_1439_Dropout_0_25_512_16_3_5_7_Mies_Van_Der_Rohe.html)

## Intent

This project, in many ways a follow-up on [word.camera](https://word.camera), was created on the shoulders of two spectacular open source contributions by [Andrej Karpathy](https://github.com/karpathy): [NeuralTalk2](https://github.com/karpathy/neuraltalk) and [Char-RNN](https://github.com/karpathy/char-rnn), both of which run in [Torch](http://torch.ch/). The code I've provided in this repository is a modest Python wrapper for a few of Karpathy's scripts, and a means to experiment with a few models that I've trained on [Nvidia K80 GPUs](http://www.nvidia.com/object/tesla-k80.html) using the High Performance Computing facilities at NYU.

In my research, I am developing tools that I hope will serve to augment human creativity. These are the first neural network models to emerge from my explorations, and I've decided to make them available to others:

    $ wget https://s3.amazonaws.com/rossgoodwin/models/2016-01-12_char-rnn_model_01_rg.t7
    $ wget https://s3.amazonaws.com/rossgoodwin/models/2016-01-12_char-rnn_model_02_rg.t7
    $ wget https://s3.amazonaws.com/rossgoodwin/models/2016-01-12_neuraltalk2_model_01_rg.t7

NOTE: These models are licensed under the Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License. That means if you use them you must attribute me (Ross Goodwin), you cannot use these models for commercial purposes, and any derivative work you produce must be licensed under the same terms.

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br /><span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Dataset" property="dct:title" rel="dct:type">NeuralSnap Models</span> by <a xmlns:cc="http://creativecommons.org/ns#" href="http://rossgoodwin.com" property="cc:attributionName" rel="cc:attributionURL">Ross Goodwin</a> are licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.<br />Based on a work at <a xmlns:dct="http://purl.org/dc/terms/" href="https://github.com/rossgoodwin/neuralsnap" rel="dct:source">https://github.com/rossgoodwin/neuralsnap</a>.

## How It Works

[NeuralTalk2](https://github.com/karpathy/neuraltalk) uses [convolutional](https://en.wikipedia.org/wiki/Convolutional_neural_network) and [recurrent](https://en.wikipedia.org/wiki/Recurrent_neural_network) neural networks to caption images. I trained my own model on the [MSCOCO dataset](http://mscoco.org/), using the general guidelines Karpathy outlines in his documentation, but made adjustments to increase verbosity.

I then trained a recurrent neural network using [Char-RNN](https://github.com/karpathy/char-rnn) on about 40MB of (mostly) 20th-century poetry from a variety of writers (and a variety of cultures) around the world.

In the output examples above, the red text is the image caption, and the poetry-trained net generated the rest of the text. The stanzas iterate through different RNN "temperature" values, an input that controls the riskiness of the model's predictions. Lower temperature results will be more repetitive and strictly grammatical, while higher temperature results contain more variety but may also contain more errors.

## How To Run It

I have tested this software on Ubuntu 14.04 and 15.10. You can try other OS options at your own risk -- in theory, it should run on anything that can run Torch and Python, although I've heard that [NeuralTalk2 does not play nice with the Raspberry Pi](https://github.com/SaMnCo/docker-neuraltalk2).

You'll need to clone the [NeuralTalk2](https://github.com/karpathy/neuraltalk) and [Char-RNN](https://github.com/karpathy/char-rnn) repos into the main folder, then follow Karpathy's README instructions to install the dependencies for NeuralTalk2 -- you don't need to worry about any of the GPU or training stuff, unless you want to use your own models. (The models I've provided are calibrated to run on CPUs.) Thankfully, the dependencies of Char-RNN are a subset of those required for NeuralTalk2.

## Usage

    $ python neuralsnap.py <output_title> <ntalk_model_filepath> <rnn_model_filepath> <image_folder_filepath>

e.g.

    $ python neuralsnap.py testing123 /path/to/neuraltalk2/model/2016-01-12_neuraltalk2_model_01_rg.t7 /path/to/char-rnn/model/2016-01-12_char-rnn_model_02_rg.t7 /path/to/image/folder

## Software License Information

As noted above, my trained models are available under the Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License. However, I have licensed the code in this repository under GPLv3.

    NeuralSnap image-to-text poetry generator
    Copyright (C) 2016  Ross Goodwin
    
    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.

    You may contact Ross Goodwin via email at ross.goodwin@gmail.com or
    address physical correspondence to:

    Ross Goodwin c/o ITP
    721 Broadway, 4th Floor
    New York, NY 10003