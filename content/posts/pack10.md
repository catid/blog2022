+++
title = "Pack10"
date = "2022-05-30"
description = "A new way to compress 16-bit raw video"
+++

## Motivation:

When streaming RGBD depth data, or when streaming Infra-red (IR) video, the input data is 16-bit instead of 8-bit.  Otherwise the frames of video arrive at 30 FPS or higher framerate, and HD resolution.

If the full 16-bit data is required, then we cannot use typical video compressors like H.264 since they are only 8-bit, and 16-bit file formats like PNG or TIFF have very poor compression ratio.  There are some formats like FLIF and JPEG-XL that have decent compression ratio for 16-bit lossless data, but also run extremely slow.  In practice, lossless is not necessary for this type of data because the sensor itself has noise and perfect quality is not required for most applications including machine learning training, so a lossy compressor would be beneficial to produce even smaller files than FLIF and JPEG-XL while also potentially running in real-time.

This large data rate makes it challenging to compress in real-time, and the large size makes it expensive to do using CPU or GPU software.  So, leveraging dedicated video encoder hardware would be preferable.  On modern graphics cards, CPUs, and high-end SBCs like the Jetson Nano, H.265 video encoder hardware can be leveraged.

One under-appreciated feature of the H.265 video encoders is that they all support MAIN10 mode, which is a 10-bit encoder.  That doesn't support 16-bit data directly, but we can design an algorithm that packs our 16-bit video frames so that they can work with a 10-bit encoder.  Since the hardware is doing most of the work, the CPU cost is minimal.


## Failures:

Initially I tried just quantizing the data down to 10-bit by finding the min/max value in the image and rescaling the input data to fit.  However, too much data is lost in the rescaling for several reasons.  Mainly, the input data had wider dynamic range than 10-bit, so some data was lost just in the packing step.  Further, the process of amplifying the decoded values also amplified error introduced by the encoder.

This approach was too sensitive to stuck pixels on the IR sensor by itself, and inspired us to do some image smoothing prior to encoding.

I noticed that video encoders perform much worse on data that has larger steps between pixels, so data with lower dynamic range that was scaled up to 10-bit also performed worse than data that was not scaled at all.

I considered using a histogram compression approach to try to preserve the pixel values for data, using Global Histogram Equalization (GHE).  However, the encoder is fundamentally lossy, and this approach would lead to larger errors for small errors in the encode.

I considered also using some of the UV channels of the image to encode data, however these are at quarter scale so there are not enough UV pixels to store information for each of the Y channel pixels.

This led to the algorithm that finally worked well.


## Quick Detour: Image Smoothing

When we look at imagery, we care about the edges but not the smooth areas.  Noise in smooth areas usually is distracting and is harder to compress.

This is going to be sensor-dependent, but generally some of the low bits of the sensor data are noisy and difficult to compress.  I ended up applying an edge-aware smoothing filter, which performs a smoothed median filter to remove stuck pixels and reduce noise away from edges.

In Halide-lang it looks like this:

```
    Expr mid3(Expr a, Expr b, Expr c) {
        return max(min(max(a, b), c), min(a, b));
    }

    ...

        Func input_repeat = BoundaryConditions::repeat_edge(input);

        // Median filter:

        max_x(x, y) = max(input_repeat(x - 1, y), input_repeat(x, y), input_repeat(x + 1, y));
        min_x(x, y) = min(input_repeat(x - 1, y), input_repeat(x, y), input_repeat(x + 1, y));
        mid_x(x, y) = mid3(input_repeat(x - 1, y), input_repeat(x, y), input_repeat(x + 1, y));

        min_max(x, y) = min(max_x(x, y - 1), max_x(x, y), max_x(x, y + 1));
        max_min(x, y) = max(min_x(x, y - 1), min_x(x, y), min_x(x, y + 1));
        mid_mid(x, y) = mid3(mid_x(x, y - 1), mid_x(x, y), mid_x(x, y + 1));

        median_image(x, y) = cast<float>( mid3(min_max(x, y), max_min(x, y), mid_mid(x, y)) );

        // Narrow blur:

        blur1_x(x, y) = (median_image(y - 1, x) + median_image(y, x) * 2.f + median_image(y + 1, x) + 2.f) / 4.f;
        Expr blur1_y = (blur1_x(y - 1, x) + blur1_x(y, x) * 2.f + blur1_x(y + 1, x) + 2.f) / 4.f;

        smoothed_boring(x, y) = saturating_cast<uint16_t>( blur1_y );

        // Interest calculation:

        const int radius1 = static_cast<int>( radius1_ );

        std::vector<float> gaussian1, gaussian2;
        build_normalized_gaussian_derivative(radius1, sigma, gaussian1, gaussian2);

        const int diameter1 = static_cast<int>( gaussian1.size() );
        Buffer<float> G1(diameter1), G2(diameter1);

        r1 = std::make_unique<RDom>(-radius1, diameter1);
        mu1_x(x, y) = sum(G1(radius1 + *r1) * cast<float>(input_repeat(x + *r1, y)));
        mu1_y(x, y) = sum(G1(radius1 + *r1) * cast<float>(input_repeat(x, y + *r1)));
        mu1_xy(x, y) = sum(G2(radius1 + *r1) * mu1_x(x, y + *r1));
        mu1_yx(x, y) = sum(G2(radius1 + *r1) * mu1_y(x + *r1, y));

        // Use Chessboard distance metric because it is the most rotationally invariant
        Expr mag = max(abs(mu1_xy(x, y)), abs(mu1_yx(x, y)));

        // Square to make the filter response more sharp
        Expr mag2 = mag * mag;

        // Normalize
        interest_image(x, y) = mag2 / (mag2 + epsilon);

        // Lerp between the denoised and contrast enhanced images,
        // based on the local interest.  Choose denoised for less interesting
        // pixels, and choose contrast enhanced for more interesting pixels.

        output_enhanced(x, y) = saturating_cast<uint16_t>(
            interest_image(x, y) * input(x, y) +
            (1.f - interest_image(x, y)) * smoothed_boring(x, y) + 0.5f );
```

16-bit lossless-compressed files become about 30% smaller after applying this smoothing kernel, while looking visually better, and it also eliminates stuck pixels.


## Introducing the Pack10 Algorithm:

Pack10 accepts a monochrome 16-bit raster image, and produces a 10-bit image that can be compressed using an H.265 MAIN10 encoder.  It has three stages: (1) Subtract smallest (2) Split high/low bits (3) Fold low bits.

The first step is to find the smallest value in the image, and then subtract all pixel values by this smallest value.  This eliminates as much energy as possible from the image before we start, without applying any rescaling that can cause problems for the encoder.

The second step is to double the height of the output image.  We will store the high 10 bits of the 16-bit input in the top half, and the low 10 bits in the bottom half.

This means that some of the bits of each pixel are repeated, but this is intentional!  The idea is that the low bits are useless if we lose any of the high bits.  By packing  an extra 4 bits at the bottom of the high bits we are adding a large +/- 16 buffer for error in the video encode, and in practice at high quality encodes this is sufficient.  Especially considering that the high bits of the pixel tend to change much slower across the image.

The final step is to fold the low bits to avoid sharp transitions due to the truncation.  For example, if the input data goes: 1022, 1023, 0, 1, 2... This will compress poorly with a video encoder because these sharp transitions spread energy across the whole DCT/DST and get quantized heavily leading to ringing artifacts around these edges.  So to greatly improve the quality, we fold these so that the above sequence becomes: 1022, 1023, 1023, 1022, 1021...  This step cuts the average pixel error about in half and is very important.

The folding process in pseudo-code is simply:

```
    if ((high_pixel_value & 1) == 1) {
        low_pixel_value = 0x3ff - low_pixel_value;
    }
```

And that's it!  Now you have a double-height image that can be encoded with H.265 MAIN10 profile.  The reverse operation is performed after the video is decoded to retrieve the full 16-bit pixel values.

Importantly, we ignore the low 4 bits of the upper half of the image, and instead copy those bits from the high 4 bits of the lower half of the image since they are less likely to be corrupted there by the video encoder.


## Results:

I cannot share the Anduril Industry test set of IR imagery that we used for evaluating the pack10 software, but I can share the results.

In practice, 2.1GB of 16-bit PNG images are compressed to a single 60 MB video file using x265, at quality sufficient to be used for machine learning training.  A 35x compression ratio is pretty good, and can be improved further using a hardware encoder.

The images are visually similar to the raws, differing mainly due to the smoothing process that eliminates a lot of sensor artifacts we do not care about anyhow.


## Fast Implementation:

For Anduril Industries, I quickly wrote up a Halide-Lang kernel that implements these operations and used the CPU auto-optimizer.  This took about 15 minutes.

```
Halide pack10: 0.12553 msec per call
Ref pack10: 0.208598 msec per call

Halide unpack10: 0.045782 msec per call
Ref unpack10: 0.150448 msec per call
```

So you can expect about a 2-3x speedup by using thread and vector optimizations.

My Halide kernel for packing:

```
        // Find range of input

        RDom r(0, picture.width(), 0, picture.height());

        Expr range_start = cast<uint32_t>( minimum(picture(r.x, r.y)) );

        output_range_start() = range_start;

        // Scale pixel values

        Expr biased_pixel = cast<uint16_t>( picture(x, y) - range_start );

        // Separate low/high bits

        Expr hi = biased_pixel >> 6;
        Expr lo = biased_pixel & 0x03ff;

        // Fold low bits to avoid sharp transitions that compress poorly
        lo = select((biased_pixel & (1 << 10)) == 0, lo, 0x3ff - lo);

        // Write output

        Expr picture_height = picture.height();

        if (in_high_bits) {
            output_packed_y_lo(x, y) = lo << 6;
            output_packed_y_hi(x, y) = hi << 6;

            output_packed_u(x, y) = cast<uint16_t>( 0x0200 << 6 ); // black
            output_packed_v(x, y) = cast<uint16_t>( 0x0200 << 6 ); // black
        } else {
            output_packed_y_lo(x, y) = lo;
            output_packed_y_hi(x, y) = hi;

            output_packed_u(x, y) = cast<uint16_t>( 0x0200 ); // black
            output_packed_v(x, y) = cast<uint16_t>( 0x0200 ); // black
        }
```

My Halide kernel for unpacking:

```
        Expr picture_height = packed_y.height() / 2;

        Expr lo = packed_y(x, y + picture_height);
        Expr hi = packed_y(x, y);

        if (in_high_bits) {
            lo = lo >> 6;
            hi = hi >> 6;
        } else {
            lo = lo & 0x3ff;
            hi = hi & 0x3ff;
        }

        // Unfold low bits to avoid sharp transitions that compress poorly
        lo = select((hi & (1 << 4)) == 0, lo, 0x3ff - lo);

        // Combine hi/lo bits

        Expr biased_pixel = ((hi >> 4) << 10) | lo;
        Expr pixel = cast<uint16_t>( biased_pixel + range_start );

        output_unpacked(x, y) = pixel;
```


## MAIN10 Encoder Primer:

Across the encoders I have used, a constant Qp of 10 is sufficient to provide a good trade-off between file size and near-lossless quality.

The H.265 file format supports "Prefix SEI" NALUs that can contain additional user-defined metadata.  I leveraged this feature to embed additional data into each frame of video that aids in decoding the imagery.  Since I already have a good investment in video bitstream parsing, this was easier than using `libheif` or some other container tool to package JSON or some other metadata along with the H.265 video file.  For your application, simply providing the metadata along-side the video file may be sufficient.  I've included the SEI prefix details here in case you're interested.

The x265 software encoder expects the 10-bit data to be provided as 16-bit values with the low 10 bits containing the pixel values.  These settings work well, disabling adaptive quantization and providing constant quality across the image:

```
    x265_param param;
    Api->param_default_preset(&param, "medium", "fastdecode");

    ... Other parameters here

    param.vui.bEnableVideoFullRangeFlag = 1;
    param.bIntraRefresh = 0;
    param.internalCsp = X265_CSP_I420;
    param.internalBitDepth = 10;
    param.rc.rateControlMode = X265_RC_CQP;
    param.rc.qp = 10;
    param.rc.aqMode = X265_AQ_NONE;
    param.rc.hevcAq = 0;

    Api->param_apply_profile(&param, "main10");
```

When encoding, you can provide any metadata as SEI using the `userSEI` parameter of the encoder:

```
    const int payload_bytes = 4 + 4 + 4;
    uint8_t payload[payload_bytes];
    WriteU32_LE(payload, 0xca7dca7d); // magic
    WriteU32_LE(payload + 4, my_metadata.RangeStart);
    WriteU32_LE(payload + 8, 0/*reserved*/);

    x265_sei_payload payloads[1];
    payloads[0].payloadSize = payload_bytes;
    payloads[0].payload = payload;
    payloads[0].payloadType = SEIPayloadType::USER_DATA_UNREGISTERED;

    PictureIn.userSEI.numPayloads = 1;
    PictureIn.userSEI.payloads = payloads;
```

The Jetson hardware encoder expects the 10-bit data to be provided as 16-bit values with the high 10 bits containing the pixel values.

These settings provide similar performance as the x265 settings above:

```
    Encoder->setProfile(V4L2_MPEG_VIDEO_H265_PROFILE_MAIN10);
    Encoder->setConstantQp(10);
    Encoder->setHWPresetType(V4L2_ENC_HW_PRESET_SLOW);
```

How to use the Jetson video encoder and how to parse H.265 and so on is outside of the scope of this article.

While encoding on the Jetson platform you can embed the prefix SEI yourself while receiving the output frames from the encoder:

```
    // SEI payload
    const int rbsp_bytes = 33;
    uint8_t rbsp_data[rbsp_bytes] = {
        0x4e, 0x01, 0x05, 0x1c, 0x2c, 0xa2, 0xde, 0x09, 0xb5, 0x17, 0x47, 0xdb, 0xbb, 0x55, 0xa4, 0xfe, 0x7f, 0xc2, 0xfc, 0x4e,
        0x7d, 0xca, 0x7d, 0xca, // magic
        0x00, 0x00, 0x00, 0x00, // RangeStart
        0x00, 0x00, 0x00, 0x00, // reserved
        0x80
    };

    // Fill in metadata
    WriteU32_LE(rbsp_data + rbsp_bytes - 9, my_metadata.RangeStart);

    // Convert to NAL unit (with emulation prevention)
    std::vector<uint8_t> nalu;
    nalu.push_back(0);
    nalu.push_back(0);
    nalu.push_back(1);
    AddPayloadEmulationPrevention(rbsp_data, rbsp_bytes, nalu);

    // Append the generated nalu to the video here
```

The ffmpeg software decoder produces 10-bit data as a 16-bit raster with the low 10 bits containing the pixel values.

To retrieve the SEI prefix from the video frames, you can parse the video bitstream and then use code like this:

```
        bool found_metadata = false;
        for (uint32_t i = 0; i < video->SeiCount; ++i) {
            dis_Sei& sei = video->SeiArray[i];
            const uint8_t* data = sei.Data;
            int bytes = sei.Bytes;

            if (bytes >= 30 && bytes <= 40) {
                std::vector<uint8_t> rbsp;
                RemovePayloadEmulationPrevention(data, bytes, rbsp);
                uint8_t* rbsp_data = rbsp.data();
                const int rbsp_size = static_cast<int>( rbsp.size() );

                if (rbsp_size == 33) {
                    const uint32_t magic = ReadU32_LE(rbsp_data + rbsp_size - 13);
                    if (magic == 0xca7dca7d) {
                        found_metadata = true;
                        my_metadata.RangeStart = ReadU32_LE(rbsp_data + rbsp_size - 9);
                        break;
                    }
                }
            }
        }
```
