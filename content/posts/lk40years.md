+++
title = "Efficient Image Alignment with Outlier Rejection Revisited"
date = "2023-08-25"
cover = "posts/lk40years/pyramids.png"
description = "Another 20 Years On"
+++

## Fast Image Alignment in 2023

Why write a paper when it can be accomplished with a blog post?

This post is about sharing my experience implementing a paper from 20 years ago: ["Efficient Image Alignment with Outlier Rejection" (Ishikawa, Matthews, Baker, 2002)](https://www.ri.cmu.edu/pub_files/pub3/ishikawa_takahiro_2002_1/ishikawa_takahiro_2002_1.pdf)

The simple and fast "classical" algorithm described by the paper still has applications today.  Since the algorithm is designed for previous generations of hardware, it can be implemented entirely on a modern CPU and still run at 200 FPS on an edge device like the Jetson Orin.  Also since the algorithm doesn't depend on a keypoint detector like e.g. SIFT features, it doesn't depend on there being detectable keypoints in the image, which is useful for scenes of just sky and moving water.

In this post I describe 4 major modern improvements to the Robust Inverse Compositional algorithm, and also 5 details that have a measurable impact on the speed or quality of image alignment.


## Foundations

The "Efficient Image Alignment with Outlier Rejection" paper presents a modification of the Lucas-Kanade Inverse Compositional algorithm.  There are a series of papers beginning with ["Lucas-Kanade 20 Years On: An Unifying Framework" (Baker & Matthews, 2003)](https://www.ri.cmu.edu/pub_files/pub3/baker_simon_2002_3/baker_simon_2002_3.pdf) that best describe this algorithm and its performance benefits.

Note that the family of Lucas-Kanade algorithms is making the assumption that the images are within about 1 pixel of correct alignment already.  What this means is that in order to align images that are farther apart, an image pyramid is used.

An image pyramid is constructed by (Gaussian) downsampling the full-resolution images to be half the size, and then repeating that operation until they are very small.  For example it could stop when the images have a dimension smaller than 32 pixels to a side.  We have to stop at about 32x32 pixels since we stop having image edges useful for alignment beyond that point.  As an example, if the input image has a size of 512x512 pixels, then the image pyramid consists of 5 images: 512x512, 256x256, 128x128, 64x64, 32x32.

Since we cannot keep downsampling forever, if the images have moved more than ~32 pixels, this algorithm is not guaranteed to produce a correct result!  This is a fundamental limitation that can be lifted by increasing the input framerate or combining it with other algorithms as discussed later.

The paper above describes an image alignment algorithm that runs fast, but ignores the effect of outliers such as objects that move between frames:

![Inverse Compositional Algorithm](ica_algorithm.png)

You can see this is an iterative algorithm that starts with some initial guess and refines the guess based on Gauss-Newton integration.  The algorithm ends when the calculated step size is below some threshold.  The result is a guess at the image warp that best aligns one image with another.

The most important step of the original algorithm is (7): The sum of `Warped Image Difference` * (element-wise product of) `Precomputed Jacobians of Template Image`.  This sum determines the step that will be taken directly.

Note the `Precomputed Jacobians of Template Image` is a series of images, calculated with variations on the theme of: `x` * `Gradient in X direction of Image at (x, y)`, `y` * `Gradient in X direction of Image at (x, y)`, and so on.  The details are not important for this discussion, but calculations like this are involved.

The implications of this step are important to understand if you want to know where the algorithm (as stated) fails.

First there are implications for floating-point precision.  For 16-bit images, the warped image difference may be as large as -65536 or more commonly closer to 0.  Furthermore the gradient is defined as the difference between neighboring pixel values, which can also be as large as -65536 or usually near 0.  Finally the Jacobian x/y values range the size of the image, which let's say is about 1000 pixels.  So multiplying all three gives a value of 4294967296000.  It's well-known that adding together large and small floating-point values loses precision, which is exactly what can happen here!  Mitigating this effect has a noticeable impact on performance as will be discussed later.

The gradient and image difference terms that are large will dominate the calculation of this sum.  Keep this in mind for the discussion on sparse optimization later.

More pressingly, both the image difference and gradient terms happen to be absolutely huge around moving objects.  For example usually the background of a scene is a very different color or brightness than foreground objects, so the edges will have large gradients.  If the foreground objects are moving around, then the image differences will be very large.  So both terms are large, and they are multiplied together!  Therefore this algorithm will unfairly weight moving objects more than most other areas of the image, making it much harder to align the images.


## "Efficient Image Alignment with Outlier Rejection"

In "Efficient Image Alignment with Outlier Rejection" the authors improve the Inverse Compositional Algorithm to be more robust to outliers:

![Robust Inverse Compositional Algorithm](rica_algorithm.png)

A mask has been added, so that some pixels no longer participate in the sum discussed in the previous section.  The pixels that have the largest image difference are masked out.  Since almost all outliers have large image differences this effectively removes these huge distracting terms from the sum.

This version of the algorithm also works on every image pixel, so it has some details about breaking up the image into tiles to make step `(8a)` less expensive, but I ended up not needing to implement this since my version selects a sparse subset of pixels already.  So, the one important improvement suggested by this paper is simply to not sum pixels that look like outliers.


## Huge Improvement: Sparsity!

As described above, the pixels with large image gradient dominate the sum.  What if we *only* summed pixels that have large image gradients and just ignored the rest?  This is the intuition that leads to a good sparse version of the algorithm.

This is inspired by the DSO paper: ["Direct Sparse Odometry" (Engel, 2016)](https://github.com/JakobEngel/dso).  The authors use low-quality but numerous feature points.  In the DSO paper the feature points are selected to be those with the maximum image gradient for NxN tiles across the image and some neighbors in a fixed pattern.

I found it sufficient to select about 1000 pixels across each template image in the pyramid that have the largest x or y gradient magnitude.  They are selected by picking a NxN tile size such that the number of selected points will be above 1000, and then for each tile selecting one pixel.

This makes being robust to moving object outliers much cheaper: As there are only about 1000 pixels selected, identifying the top 20% of pixels with the largest image difference is fast with `std::nth_element()` sorting an array of `struct { float abs_image_delta; uint16_t x, y; }` and subtracting out any gradient sum terms from those locations.  This is the same as the mask step in the Robust Inverse Compositional Algorithm, but applied to a sparse set of pixels so it's simpler.

Since the number of pixels selected for each layer of the image pyramid is the same, each layer takes about the same amount of time to process regardless of its resolution.  This also hugely improves the speed of the Inverse Compositional algorithm, while also making it trivial to reject outlier pixels.

A side note: When sampling from warped images at these sparsely selected locations, multiple image pixels have to be interpolated to determine a warped pixel value.  I found that using Lanczos2 resampling performs slightly better than linear resampling for image alignment, without affecting runtime.


## Huge Improvement: Keyframes!

Recall that there are several steps `(3)-(6)` that need to be pre-computed for the template image in the Inverse Compositional algorithm.  Instead of calculating this for every new frame, a simple optimization that saves about 25% of the runtime is to treat every other frame as the keyframe/template image.  So it instead alternates aligning forwards/backwards from a keyframe.  When aligning backwards, the only change to the algorithm is to invert the calculated transform using double precision math.  So this optimization is very easy to implement.

This can also support higher framerates by picking the keyframes to be farther apart and assuming the alignment will be sufficient.


## Huge Improvement: Better Initialization!

Normally the algorithm is initialized with an identity matrix, meaning it starts with the assumption of no displacement between the frames.  What if we used another algorithm to provide a better initialization?  This turns out to work really well.

Phase correlation is slow on full resolution but on a small image it can quickly provide an initial estimate of alignment between frames in x/y.  This complements the LK algorithm nicely, which as discussed earlier is limited to a maximum of about 32 pixels of displacement betwen frames depending on the number of pyramid layers used.

This is inspired by the Google paper ["Handheld Multi-Frame Super-Resolution" ( Wronski et al, 2021)](https://sites.google.com/view/handheld-super-res/) where the authors use exactly this hybrid combination.

OpenCV cv::phaseCorrelate() is very fast, taking less than a millisecond to provide an x/y displacement estimate.  I also implemented my own version of this with Halide and it performed just as well as the OpenCV version, but it used more CPU resources.  It's possible a version based on FFTW would perform better than the OpenCV version but I haven't tested.

In my testing, adding phase correlation is a cheap way to greatly improve the convergence speed and convergence rate of the algorithm.  Since the initialization is already close to the correct final answer, often times fewer iterations are needed, so it can run faster as well.


## Huge Improvement: Detect Failures!

The original algorithm does not include any kind of detection for failed alignment.  If failures can be detected, then drop-outs can be handled appropriately rather than producing bad results downstream.  After some fiddling I came up with an effective way to do this:

Failure detection is performed separately at each level of the image pyramid.  It accumulates the individual steps to keep track of how much the corners are being moved.  Recall that the Lucas-Kanade algorithm is designed to align images that are within about 1 pixel at each level of the pyramid.  So, if the images are being shifted by many pixels then it indicates that the algorithm is not working as expected.

Usually rotation between frames does cause multiple pixels of motion at the corners, but alignment will still succeed, and so the threshold should be more than 1 pixel.  Usually a failure to align leads to large corner displacements.

After some testing I found that 6 pixels of accumulated displacement at any of the 4 corners indicates a failure to converge.  The smallest pyramid layer can have poor initialization so its threshold should be set higher at about 16 pixels maximum.  Note these are pixels down the image pyramid, so 1 pixel at these layers represents many pixels of displacement at full resolution.

Note this is resetting the step accumulation for each layer of the pyramid, so it only detects a failure if more than 6 pixels of displacement is seen in a corner point from steps at each level.


## 2-Point, 3-Point vs 3x3 Direct Matrix Parameterization

When aligning images for the purpose of video stabilization, it is sufficient to solve for a similarity or affine transform between frames.  These transforms contain information about scale, rotation, and translation between the frames.  Affine transforms also represnt image skew as well, which can account for rolling shutter artifacts.  So, for low frequency vibration a similarity transform is fine.  For high frequency vibration, an affine transform is probably needed.

Similarity transforms can be represented by 4 parameters, such as how the x,y coordinates of the top two corners of the image are displaced by the warp.

Affine transforms can be represented by 6 parameters, such as how an additional point in the lower left is displaced.

In ["Parameterizing Homographies (Baker & Kanade, 2006)"](https://www.ri.cmu.edu/pub_files/pub4/baker_simon_2006_1/baker_simon_2006_1.pdf) Kanade returns to explain that solving directly for transform matrices performs significantly worse than solving for the 4-point corner parameterization.

So, I decided to explore if this is also the case for similarity (2-point) or affine (3-point) image alignment.  I implemented both the direct form and the representation using corner points and found that the direct form is better.  My intuition for why this might be the case goes back to the math in the previous section.

This is the function that describes how pixels are transformed by the image warp (for the similarity transform):

```
        W_x = (1+A) * x     - B * y + TX
        W_y =     B * x + (1+A) * y + TY
```

Taking the derivative with respect to each parameter (A, B, TX, TY) provides the Jacobian:

```
              A   B   TX   TY
            ===================
            [ x  -y    1    0 ]  <- W_x
        J = [                 ]
            [ y   x    0    1 ]  <- W_y
```

Without getting too far into the weeds, the Jacobian for the canonical corner 2-point representation of the similarity transform:

```
                 u1    u2      v1     v2
            ==============================
            [ 1-x/w   x/w     y/w   -y/w ]  <- W_x
        J = [                            ]
            [  -y/w   y/w   1-x/w    x/w ]  <- W_y
```

The point I want to make here is that the Jacobian for the direct form is a lot simpler.  This means that it is easier to optimize.  So it is unsurprising that the image alignment succeeds more often when using the matrix form.


## Devil is in the Details: Image Pyramid

OpenCV's cv::pyrDown() with cv::BORDER_REFLECT performs worse than a custom implementation of Gaussian pyramid using a separable filter with 5 taps:

```
    Taps = [ 1/16, 4/16, 6/16, 4/16, 1/16 ]
```

This is applied in Y and then in X.  Edges are repeated, rather than reflected.  The result is a Gaussian blur.  Then every other pixel is discarded to downsample the image.

I tried using a 3 tap filter as well, thinking that maybe a tigther blur kernel would help, but it performed worse.


## Devil is in the Details: Floating-Point Precision

While calculating the image pyramid, I tried applying a normalization factor of 1/1024, which turned out to improve the performance measurably.  Since I was using 16-bit input images, this constant makes sense for this higher bit depth data.  For 8-bit input images, probably something like 1/128 works better.

The reason for this is subtle but related to the earlier discussion about floating-point rounding errors from addition.  By dividing by 1024, the image differences are knocked down mostly below 1, and so the product of these remains small rather than exploding in that critical sum discussed in previous sections: The product of two numbers less than 1 is still less than 1.

Note that without this normalization, the 3-point parameterization discussed earlier actually performs *better* than the direct form, because the normalization is built into the Jacobian (e.g. `x/w`)!  By normalizing the direct form, it ends up performing better.  For this reason, I'm questioning if the 4-point parameter version is actually better than the direct form too as claimed by the authors.  Perhaps if normalization of the direct form was used, the results would have favored the direct form?  No idea - I haven't tested the 4-point parameterization personally.

Outside of the inner loops that need to be fast, math should be done with full 64-bit double precision.  Using 32-bit floats to represent the affine matrix yields an alignment precision of just 0.4 pixels at best due to floating point rounding errors while calculating a matrix inverse, which is insufficient for most applications.  Normally you expect accuracy of around 0.02 pixels or better for the corner point positions.


## Devil is in the Details: Image Gradients

I tested several alternative interpretations of image gradients: Central Difference, Scharr, Prewitt, Sobel, FS3, and FS5.

Central Difference performed the best, and is also the simplest:

```
Central difference in X: (Image(x+1, y) - Image(x-1, y)) * 0.5
Central difference in Y: (Image(x, y+1) - Image(x, y-1)) * 0.5
(Note the 0.5 factor is load-bearing.)
```

My intuition for why is that this tight kernel preserves details for each layer of the pyramid better than larger kernels that blur together pixels.  I believe each layer represents one narrow "frequency" of information a bit like a Fourier transform, so keeping the filters sharp keeps everything tuned to the information content of that layer.  But this again is just my gut feeling for why this works best.


## Devil is in the Details: Halide-lang or OpenCV?

When implementing each algorithm required, I evaluated Halide-lang versions written by myself and OpenCV versions.  For the most part OpenCV is faster on CPU when it supports an operation.  Since sparse sampling is used, OpenCV does not have functions for many of the steps of the algorithm.  Halide makes implementing and deploying these more complex operations entirely painless, so I recommend incorporating it into your workflow.  I shared a good example project using Halide here: https://github.com/catid/halide-test
