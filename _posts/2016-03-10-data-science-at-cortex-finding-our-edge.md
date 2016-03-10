---
author: Sourav Dey
author_twitter: "@resdntalien"
categories:
- data science
- machine learning
- algorithms
layout: post
title: "Data Science at Cortex: Finding Our Edge"
---

In our previous post we talked about finding edges in sensor signals so we could use them to help us estimate a building's start time.  We want to find sharp transitions in the various HVAC sensors -- rising edges for electricity, steam, and static pressure and falling edges for supply air temperature (SAT).  It's easy for a human to pick out edges -- but how do we teach a computer to do it?

<!--break-->

## Background Research

Turns out we're not the first ones to have this problem. Edge finding is a common problem in image processing where it is a pre-processing step to many computer vision techniques.  After some Google searching we decided to use a traditional technique called the [Canny edge finder](https://en.wikipedia.org/wiki/Canny_edge_detector). Even better, [scikit-image](http://scikit-image.org/docs/dev/auto_examples/plot_canny.html) in Python already has a built-in function for it.  

After some fiddling though, we found that we couldn't use the scikit-image implementation out of the box.  The primary problem was that the Canny edge detector is tuned to work on images rather than time series.  Specifically, the Canny algorithm finds the center of an edge, but we want to find the beginning of an edge when the signal first started to transition.  This is often the case when using textbook algorithms.  They have to be adapted to your needs.  That's ok, this is what we do and why data scientists have jobs.

## Detection Algorithm

After some trial and error, we developed what I call the adapted Canny algorithm.  The flowchart below shows the basic steps of the algorithm.

<img src="https://s3.amazonaws.com/cortex-blog-content/images/01_detection_algorithm.png">

The figure below illustrates the intermediate steps of this algorithm on some example data.  It will be easier to follow the following discussion with this diagram.

<img src="https://s3.amazonaws.com/cortex-blog-content/images/02_edge_graph.png">

The input, x[n], is the portion of the sensor's time series for which we want to find edges.  This is typically the period from 8pm to 8am. This goes into a series of steps.

1.  **Normalization.**  We normalize the time series by the historical min and max values from training.  This is shown in blue on the plot.
<img src="https://s3.amazonaws.com/cortex-blog-content/images/03_x_norm_min.png">

2.  **Gaussian Smoothing.**  We smooth the noise in the signal by convolving with a Gaussian kernel.  This ensures that we don't find false edges on noise.  See [here](https://en.wikipedia.org/wiki/Gaussian_filter) for more details.  * here represents convolution. h[n] is the Gaussian kernel. This is shown in green on the plot.
<img src="https://s3.amazonaws.com/cortex-blog-content/images/04_smooth_norm.png">

3.  **First difference.**   We take the discrete time equivalent of a derivative.  This is shown as red on the plot.
<img src="https://s3.amazonaws.com/cortex-blog-content/images/05_smooth_smooth.png">

4.  **Thresholding.**  We mark portions where the x_diff[n] is over (or under) a threshold, T_edge.   Contiguous portions where x_diff[n] exceeds the threshold are edges in the original signal.  The threshold is computed in the training step of the algorithm.  The threshold value is shown in purple on the plot. 
<img src="https://s3.amazonaws.com/cortex-blog-content/images/06_edge_otherwise.png">

5.  **Edge extraction.**  We use x_edge[n] and the original signal, x[n], to extract relevant edge information.  The extracted information is described below.

The final output is a set of edges with the following information (note: there could be multiple edges in a single signal or none at all).  

1.  Time the edge begins.  This is shown as a black vertical dotted line on the plot.

2.  Value of x[n] when the edge begins. 

3.  Time the edge ends.  This is shown as a grey vertical dotted line on the plot.

4.  Value of x[n] when the edge ends.

5.  Edge strength which is computed as the max x_diff[n] value during the edge.  This measures the "sharpness" of the transition. 

## Training Algorithm

What we've described so far is the **detection** algorithm.  It it what is run every day to find edges in all the sensor signals.  It requires three parameters, x_min, x_max, and T_edge, which must be learned from historical sensor data.  The algorithm to automatically learn these parameters is called the **training** algorithm.  This is run infrequently -- once a month or less -- whenever we want to retune the parameters.   Note that this split of the method into an **detection** algorithm and a **training** algorithm is extremely common in the data science world.  Most machine learning algorithms, from support vector machines to deep convolutional nets, have this pattern. 

The flowchart below shows the basic steps of the training algorithm.

<img src="https://s3.amazonaws.com/cortex-blog-content/images/07_training_algorithm.png">

The input, x[n], is as much historical data as we want to put into the training.  It can be all of the data, or some recent history.  This input data is fed into a series of steps: 

1.  **Min.** Find the minimum value of the historical data. This is x_min.

2.  **Max.** Find the maximum value of the historical data. This is x_max.

3.  **Normalization.**  We normalize the time series by the newly calculated x_min and x_max.  As in the detection algorithm,
<img src="https://s3.amazonaws.com/cortex-blog-content/images/08_xnorm_max_min.png">

4.  **Gaussian Smoothing.** As in the detection algorithm, we smooth the noise in the signal by convolving with a Gaussian kernel.  

5.  **First difference.** As in the detection algorithm, we take the discrete time equivalent of a derivative.

6.  **Clipping.** Depending on if we're looking for rising edges or falling edges we zero out the negative or positive values of x_diff[n].
<img src="https://s3.amazonaws.com/cortex-blog-content/images/09_clip.png">

7.  **Otsu's Method.**  This figures out the threshold to be used in the detection algorithm.  It finds the threshold by finding a binary split of x_diff[n] that maximizes the interclass variance. This is a technical way of saying that it finds a the best threshold that differentiates the peaks in the signal from the valleys. Read [here](https://en.wikipedia.org/wiki/Otsu%27s_method) for more information about Otsu's method if you're interested. 

Note that sigma, which controls the amount of smoothing in the Gaussian filter, is not an automatically learned parameter.  This is human trained.  We typically choose the smallest possible value so we smooth the noise but not the edges.  It can be different for every sensor, but for the most part we've found that it can be the same across sensors of the same type no matter what building they are in.  

So that's it. Our edge finder. The first step toward our start time estimator.  In the next post we'll take the output of this edge finder and synthesize it into an estimated HVAC start time for a building.  Before I go, I want to emphasize that this algorithm didn't just spring to life fully formed. Rather it was an iterative process driven by testing on real data.  We'll talk much more about this process in the next blog post.