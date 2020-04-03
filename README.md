# chrono-photo

Chronophotography command line tool and library in [Rust](https://www.rust-lang.org/).

![A simple Chronophotography example](https://user-images.githubusercontent.com/44003176/77975353-236da480-72fa-11ea-9ff9-5c110895fe5d.jpg)
<sup>_A simple Chronophotography example_</sup>

This tool creates chrono-photos like 
[Xavi Bou's "Ornithographies"](http://www.xavibou.com/) 
from video footage or photo series.

_Warning:_ This project is in a very experimental state.
Supports only basic image processing so far.
However, the image above shows a proof of concept for the algorithm,
based on outlier detection (see section [How it works](#how-it-works) for details). 

## Command line tool

### Installation

* Download the [latest binaries](https://github.com/mlange-42/chrono-photo/releases).
* Unzip somewhere with write privileges (only required for running examples in place).

### Usage

* Try the example batch files in sub-directory [`/cmd_examples`](/cmd_examples).
* To view the full list of options, run `chrono-photo --help`

## Library / crate

To use this crate as a library, add the following to your `Cargo.toml` dependencies section:
```
chrono-photo = { git = "https://github.com/mlange-42/chrono-photo.git" }
```

## Development version

For the latest development version, see branch [`dev`](https://github.com/mlange-42/chrono-photo/tree/dev).

## How it works

The principle idea is to stack all images to be processes, and analyze the entire stack pixel by pixel
(but see also section [Technical realization](#technical-realization)).

Given a typical use case like a moving object in front of a static background, 
a certain pixel will have very similar colors in most images. 
In one or a few images, the pixel's value may be different, as it shows the object rather than the background.
I.e. among the pixel's color from all images, these images would be outliers.

The idea now is to use these outliers for the output image, if they exist for a certain pixel, 
or a non-outlier if they don't.

### Outlier detection

Outlier detection in the current version uses multi-dimensional distance to the median,
and a threshold provided via option `--mode` (default: 0.1; `--mode outlier-0.1`). 
The threshold is relative to the per-band color range (i.e. fraction of range [0, 255] for 8 bits per color band).

A pixel value is categorized as an outlier if it's distance from the median is at least the threshold.

#### Pixel selection among outliers

If only one outlier is found for a pixel, it is used as the pixel's value.

If more than one outlier is found for a pixel, different methods can be used to select among them via option `--outlier`:
* `first`: use the first outlier found.
* `last`: use the last outlier found.
* `extreme`: use the most extreme outlier (the default).
* `average`: use the average of all outliers.

#### Pixel selection in absence of outliers

If no outliers are found for a pixel, different methods can be used to select the pixel's value via option `--background`:
* `first`: Use the pixel value from the first image.
* `random`: Use a randomly selected pixel value, selected among all images. May result in a noisy image.
* `average`: Use the average pixel value of all images. Can be used for blurring, but may result in banding for low contrast backgrounds.
* `median`: Use the median pixel value of all images. May result in banding for low contrast backgrounds.

#### Parameter selection

Finding the best options for pixel selection, as well as an outlier threshold that fits the noise in the input images,
may require some trial and error.

In addition to inspection of the produced image, use option `--outlier-output <path>` to write a black-and-white
image showing which pixels were filled based on outliers (white), and which were not (black).

If there are black pixels inside the moving object(s), the outlier threshold should be decreased. 
On the other hand, if there are white pixels outside the moving object(s), the threshold should be increased
(may happen due to too much image noise, an insufficiently steady camera, or due to motion in the background).

### Technical realization

Holding a large number of high resolution images in memory at the same time is not feasible. 

Therefore, before actual processing, the time-stack of images with (x, y) coordinates
is converted into a number of temporary files, each containing data in (x, t) coordinates.

For example, the first temporary file contains the first row of pixels from each image.

Using these temporary files, all images can be processes row by row, without overloading memory, as explained above.