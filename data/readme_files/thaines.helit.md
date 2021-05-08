# Helit
This is basically a dumping ground for the code that I develop for my research. I am a firm believer that all source code must be published, for the exact same argument that you must publish. Saying this I often omit the code that generated a papers results, simply because I don't have the bandwidth to distribute the data sets or the time to clean up what is often a horrific mess. Regardless, the actual algorithm is always available for people to run their own experiments with, and if you request testing code from me I'll probably provide it under the condition that your not allowed to laugh. It is all implemented using Python, but some modules also contain C/C++ code for the bits that need to be fast, either inline via scipy.weave or as a proper Python modules that need to be compiled. In both cases as long as Python has access to a compiler the code should be automatically compiled when first run (this is reliable on Linux, but can take some setup/fiddling before it works on Mac/Windows). A few modules can also be installed via the standard setup.py mechanism. You may find my personal website at http://thaines.com

I tend to use a variety of licenses, typically BSD, Apache and GPL - check the headers. Utility stuff is usually BSD, implementations of other peoples algorithms Apache (Roughly summarised as BSD but with a longer license!) and my own research code Apache or GPL, depending on mood/project constraints. Its mostly Apache 2. I do licensing on a directory by directory basis - you will not find differently licensed files within a single directory, and the dependencies between directories obviously follow the relevant requirements. If you have good cause to request the code in another license feel free to ask - its not like I am attempting to make money out of this, so if its reasonable I will say yes.

It should be noted that I develop exclusively on 64 bit Linux, but being Python most of this code should just work on other platforms. scipy and OpenCV (multiple versions!) are also used - most if not all modules use scipy, whilst OpenCV is mostly used by the test scripts rather than the modules themselves. Installing either of these libraries should be easy on mainstream platforms. scipy.weave is used by some modules, and is a little tricky to install, as it requires a C++ compiler to be available. Whilst it works out of the box on most Linux installs, assuming you have a compiler, it can require some effort to get working elsewhere. Its not particularly hard under Windows - just google the instructions, but I have heard that it is a total pita if you are using a mac. You might want to investigate running Linux inside a virtual machine instead (I've successfully done this with virtual box - http://www.virtualbox.org/), as it is just easier. Whilst I'm 90% sure that the code should work on 32 bit operating systems there is a risk that I have missed something, as I never test on something so antiquated. Also, I use soft links in some cases, which I don't think Windows supports - easiest solution is to duplicate the data; alternatively you can add the root directory to Python's path environment variable. Or if the code supports it, just install it with setup.py

This code is primarily for research, which is to say it is not of industrial standards. Some of it is sufficiently neat and robust that it would probably pass the needed testing regime however, once you slapped some sanity checking code in. Some of it is not however - depends mostly on how far out the next deadline was! Regardless, do report bugs to me so I can try to find the time to fix them. I'll also consider feature requests, so don't be afraid to ask.

If you use this code for your own research then it would be great if you could cite me. Much of the code is for specific papers - that will usually be given in the description, so the paper it was written for would be the obvious choice. When that is not the case, as in the code is stand alone, a competitor I was testing against or for a failed research project (!), then something like:
```
@MISC{helit,
  author = {T. S. F. Haines},
  title = {\url{https://github.com/thaines/helit}},
  year = {2010--},
}
```
If you're doing cool stuff with my code - industry, research, personal project - it does not matter, then I would love to hear about it:-)

If anyone wants to contact me for any of the above reasons, or any other good reason, then my email address is [x]@[y].com where [x]=thaines and [y]=gmail


Current contents (alphabetical):

**ddhdp**: (GPL 3.0) My Delta-Dual Hierarchical Dirichlet Processes implementation, from the paper 'Delta-Dual Hierarchical Dirichlet Processes: A pragmatic abnormal behaviour detector' by T. S. F. Haines and T. Xiang (Can be obtained from my website.). It shares a lot of code with the dhdp implementation, unsurprisingly, making similarly extensive use of scipy.weave. Also, if dhdp is complex this is bordering on the insane - its an awful lot of very complex code, compliments of a lot of variables that need to be Gibbs sampled.

**ddp**: (Apache 2.0) Discrete dynamic programming. Nothing special.

**df**: (Apache 2.0) A decision forest (random forest) implementation. It is extremely modular, for future expandability, but rather limited. Very flexible, with support for incremental learning and both continuous and discrete features. Currently only supports classification with a limited set of test generation techniques, which is basically the standard feature set for a typical random forest implementation. Currently it is Python with numpy/scipy only; code is well commented and should be relatively easy to understand.

**dhdp**: (Apache 2.0) A Dual Hierarchical Dirichlet processes implementation, using Gibbs sampling. Also includes the ability to switch off the document clustering and obtain a HDP implementation. Its rather complex - a lot is going on, and its design is arguably not the best. Makes extensive use of scipy.weave as most of the code is actually in C++.

**dp_al**: (GPL 3.0) An active learning method of mine, that uses Dirichlet processes. The relevant paper is 'Active Learning using Dirichlet Processes for Rare Class Discovery and Classification' by T. S. F. Haines and T. Xiang, from BMVC 2011 (Can be obtained from my website.). A very simple algorithm that works very well when faced with the competing goals of finding new categories, particularly rare ones, and refining the classification boundaries of existing categories. It is a bit absurd me putting code up, given that implementation is so easy, so the main value here is the test code and the framework that allows for comparison with a bunch of other active learning algorithms that are also implemented within this module.

**dpgmm**: (Apache 2.0) A Dirichlet process Gaussian mixture model, implemented using the mean field variational technique. Its about as good as a general purpose density estimator can get, though it suffers from a heavy computational and memory burden. Code is in pure python, depends on the gcp module and it is very neat and reasonably well commented - speed is reasonable for the method as the code vectorises well. Unlike some other implementations handles incremental learning correctly - both adding extra sticks to the model and adding extra data after convergence. This code was used for: https://arxiv.org/abs/1801.08009 - that or my background subtraction journal make sense to cite if you use this code.

**dp_utils**: (BSD) A collection of C++ code for handling Dirichlet processes. Used by other modules in the system (dhdp and ddhdp).

**frf**: (Apache 2.0) A standard random forest implementation, all in C and designed for speed, with good I/O and support for multi-processing plus all the usual bells and whistles. Was created for the paper 'My Text in Your Handwriting' when I got frustrated with the scikit learn implementations horrible I/O speeds.

**gbp**: (Apache 2.0) Gaussian belief propagation implementation. Also does Gaussian TRW-S, - make of that what you will! This is really just a simple linear solver, but with an interface that is really helpful for solving certain problems. If your model is a chain then its Kalman smoothing. Interface is fairly advanced, and will allow you to solve problems, modify them, then solve them again, quicker because you're already close. It was the core approach of my paper 'Integrating Stereo with Shape-from-Shading derived Orientation Information', though this is a reimplementation done for my more recent paper, 'My Text in Your Handwriting'. Includes a script for solving sparse linear problems and another for removing curl from normal maps.

**gcp**: (Apache 2.0) The code associated with my Gaussian conjugate prior cheat sheet. Basically an implementation of exactly what you would expect, though it has a few extra features/tweaks since I uploaded it as an archive to download from my website due to it seeing some real world usage. It of course has a conjugate prior class, with the ability to provide new samples and draw a Gaussian or draw a Student-t distribution that is the probability of a sample drawn from the Gaussian drawn from the prior, but with the Gaussian integrated out. Additionally classes to represent Gaussian, Wishart and Student-t distributions are provided, as well as an incremental Gaussian fitter. All written in pure python; actually fast, due to using eigensum!

**gmm**: (Apache 2.0) A fairly standard Gaussian mixture model library, that uses k-means for initialisation followed by EM to fit with the actual model. Only supports isotropic kernels, but then non-isotropic kernels can cause stability issues with EM. Has a Bayesian information criterion (BIC) implementation, for automatically selecting the number of clusters. Includes some very nice, and particularly fast, k-means implementations, in case that is all you want. Makes extensive use of scipy.weave.

**graph_cuts**: (Apache 2.0) Just a max flow implimentation and an interface for doing binary labelings on nD grids.

**handwriting**: (Mixed licenses) All of the code specific to the paper 'My Text in Your Handwriting', by T. S. F. Haines (me), O. Mac Aodha and  G. J. Brostow.

**hg**: (Apache 2.0) Simple module for constructing and applying homographies to images, plus some related gubbins.

**kde_inc**: (Apache 2.0) A simple incremental kernel density estimation module with Gaussian kernels, that uses a fixed number of kernels and greedily merges kernels together when it exceeds that cap. Also includes leave one out optimisation of the kernel size, in terms of a symmetric precision matrix. Nothing special really - I only implemented this so I could match the experimental setup of another paper. Its simple and neat however, and I guess could be useful when the fixed time property matters, though if that is really the case you would probably want a C version, rather than this python-only snail.

**lda_gibbs**: (Apache 2.0) A latent Dirichlet allocation implementation using Gibbs sampling. Written as a learning exercise and so not the best organised code, but fairly well commented, so has possible educational value. Uses scipy.weave and is hence reasonably fast.

**lda_var**: (Apache 2.0) A latent Dirichlet allocation implementation using the mean field variational method. Again, written as a learning exercise. Code is disturbingly neat, plus short, and probably quite educational. It is in straight python without any C++, so it is easy to run - it could of course be faster using inline code, but still manages to be an order of magnitude faster than the Gibbs approach, which does use C++.

**misc**: (Apache 2.0) Assorted miscellaneous things - a dumping ground for things that are not large enough to justify a module of their own.

**ms**: (Apache 2.0) Provides a mean shift implementation, but also includes kernel density estimation and subspace constrained mean shift using the same object, such that they are all using the same underlying density estimate. Also has the multiplication method from the "non-parametric belief propagation" paper, fully generalised to work with all possible kernels - this code makes it trivial to do belief propagation over continuous data with complex kernels. Includes multiple spatial indexing schemes and kernel types, including multiple options for different kinds of directional data, and the ability to combine kernels. For instance, you can have a Gaussian on position and a mirrored Fisher on rotation (as a Quaternion) if doing density estimation on the position and rotation of objects. You can even then do belief propagation on relative object positions using the multiplication capability. Clustering is supported, with a choice of cluster intersection tests, as well as the ability to interpret exemplar indexing dimensions of the data matrix as extra features, so it can handle the traditional image segmentation scenario sensibly. Can do on the fly conversion of data types, so you can have an angle in radians in the feature vector that is converted to a unit vector with a Fisher distribution over it, but only internally, so its returned to the user as an angle, for instance. This module is all in C, and has a setup.py file so it can be installed. Its also one of my most heavily engineered modules, with loads of test files (some of which create really pretty outputs;-) ). The code could probably be used in production, and was used by my paper 'My Text in Your Handwriting'.

**p_cat** (Apache 2.0) A bunch of probabilistic classifiers, using some of the density estimation methods also in this code base. Does it all via a standard interface, so they can be swapped out, and includes incremental learning and, somewhat unusually, the probability of a sample belonging to an unknown class, as calculated under a Dirichlet process assumption (this last feature is for a paper).

**ply2** (BSD) My extension of the ply file format, for more general use. Includes a specification and pure Python reading/writing code. Intention is for this to be a good format to fill the gap between json and hdf5, particularly for large quantities of human readable data. Was created for the handwriting project, so particular attention has been made to string support. Done without the permission of the original authors, so I hope they don't mind!

**rlda**: (GPL 3.0) My region LDA implementation - see the paper 'Video Topic Modelling with Behavioural Segmentation' by T. S. F. Haines and T. Xiang (Downloadable from my website.). A topic model that includes a behavioural segmentation within the same model, which is simultaneously solved. Designed to be good at analysing traffic data by virtue of it inferring the regions of the input where each activity occurs, giving it a better generalisation capability.

**smp**: (GPL 3.0) This Sparse Multinomial Posterior library solves the rather unusual problem of estimating a multinomial given draws from it, when those draws are sparse. That is, when the counts for some of the entries are missing. Generates the expected value of the multinomial being estimated. Uses scipy.weave.

**svm**: (Apache 2.0) A support vector machine implementation, which only supports the traditional classification scenario. It is however implemented using some of the best available methods, and includes leave one out model selection with a decent set of kernel types and various useful defaults for when you just want to throw data at it. Its not as optimised as it could be, but is still plenty fast. I know there are already various other libraries for doing this with Python, but my testing showed serious issues with all of them, specifically strange restrictions, lack of multiclass support and an inability to save models etc, not to mention absolutely dire interfaces and little documentation. This is only from a Python point of view however - if you don't care which language you use then there are better libraries. Also, the above is a description of a historic problem, which is no longer the case.

**swood**: (Apache 2.0) My old random forest implementation - perfectly functional, but limited, and was created as a learning exercise. I would recommend using one of my other implementations instead.

**utils**: (BSD) Basic utilities used by almost all of my modules. Most useful are the progress bar class, and the make code, that means all my Python C modules automatically compile - great when doing research as forgetting to run make is no longer a problem! Unusual stuff includes python code to change a python programs process name (Good for those killall's), a function that adds line numbers to scipy.weave code, which is essential if you want to debug them, a multiprocess map replacement, and code for generating documentation.

**utils_gui**: (BSD) An image viewer for GTK 3. Works using a tile concept (for speed) and supports efficient zooming, absurdly large images and using a tablet to control it.

**video**: (GPL 3.0) A node based system for processing video, that includes various computer vision algorithms, e.g. optical flow, background subtraction. This last one is my own algorithm: 'Background Subtraction with Dirichlet Processes' by Tom SF Haines & Tao Xiang, ECCV 2012. Uses OpenCV and scipy; also compiles C code and uses OpenCL when available. 