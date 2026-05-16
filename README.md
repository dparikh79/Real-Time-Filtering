# Real-Time Video Filtering in C++ with OpenCV

A live webcam pipeline that applies thirteen image filters to a video stream in real time, all written from scratch in C++. I implemented the convolution kernels, Sobel operators, gradient magnitude, color quantization, cartoonization, and a Kelvin photo filter directly against `cv::Mat` rather than calling the equivalent OpenCV one-liners, so the project is as much about understanding what the library does internally as it is about producing the output.

Built as Project 1 of **CS 5330: Pattern Recognition and Computer Vision** at Northeastern University.

## Why

OpenCV makes it easy to call `cv::GaussianBlur` or `cv::Sobel` and move on. That hides what is actually happening to every pixel. The goal here was to write each filter as a separable convolution against the raw `Vec3b` buffer, prove I could hit interactive frame rates on a laptop webcam, and stack the filters into something that looks like a real photo effect (the cartoonizer and the Kelvin LUT in particular).

## Demo

Two recorded sessions live in the repo root, captured straight from the running program:

- `Video Result 1.mp4` (37 MB)
- `Video Result 2.mp4` (60 MB)

A full write-up with side-by-side stills and notes on each filter is in `Project Report.pdf`. Sample stills captured by pressing `s` during a run live in `Images from Video Stream/`.

## Filters Implemented

All thirteen run live against the webcam feed. Each one is its own keypress, can be toggled on or off independently, and can be stacked with the others.

| Key | Filter | Notes |
|-----|--------|-------|
| `g` | Grayscale (OpenCV `cvtColor`) | Baseline reference |
| `h` | Grayscale (custom) | Hand-rolled BGR weighted sum, `0.114 B + 0.587 G + 0.299 R` |
| `b` | Gaussian Blur 5x5 | Separable filter, horizontal pass then vertical pass, `[1 2 4 2 1] / 10` |
| `x` | Sobel X 3x3 | Separable, signed 16-bit intermediate (`CV_16SC3`) |
| `y` | Sobel Y 3x3 | Same shape as X, transposed kernel |
| `m` | Gradient magnitude | `sqrt(Sx^2 + Sy^2)` per channel |
| `i` | Blur + quantize | 5x5 blur followed by N-level color quantization (default 7) |
| `c` | Cartoonize | Blur-quantize plus a magnitude-thresholded black edge overlay |
| `n` | Negative | `255 - pixel` per channel |
| `a` | Pencil sketch | Inverted blurred grayscale dodged against the original |
| `w` | Sharpen | 3x3 unsharp kernel via `filter2D` |
| `k` | Kelvin photo filter | Per-channel lookup table built from a 5 to 6 point interpolated curve |
| `s` | Save | Snapshots every active window to disk as `Image_*.jpg` |
| `q` | Quit | |

A separate program (`colorTransfer.cpp`) implements Reinhard-style color transfer between two still images in the Lab color space.

## Quickstart

Requires `g++`, OpenCV 4, and `pkg-config`. A webcam (`/dev/video0` on Linux, default capture device on macOS or Windows) is mandatory for the live demo.

```bash
cd "Project Files"
make
./vidDisplay
```

The program prints the keymap on startup and waits for a keypress before starting the capture loop. Press the filter keys above to toggle windows, `s` to snapshot, `q` to quit.

To build the still-image color transfer demo:

```bash
g++ colorTransfer.cpp -o colorTransfer `pkg-config opencv4 --cflags --libs`
./colorTransfer
```

## Architecture

```
Project Files/
  filter.hpp          # All filter kernels (header-only, inlined)
  vidDisplay.cpp      # Live capture loop + window/key state machine
  colorTransfer.cpp   # Lab-space mean/std-dev color transfer (still images)
  imgDisplay.cpp      # Minimal "load and view an image" harness
  Makefile            # pkg-config opencv4 one-liner
  Scenery3.jpg        # Test image for colorTransfer
  Scenery4.jpg        # Test image for colorTransfer
Images from Video Stream/   # Snapshots captured during demo runs
Video Result 1.mp4          # Recorded demo
Video Result 2.mp4          # Recorded demo
Project Report.pdf          # Full write-up with stills + analysis
```

The video loop in `vidDisplay.cpp` is a single-threaded state machine. Each frame: read from `VideoCapture`, check which filter flags are set, run those filters into pre-allocated `cv::Mat` buffers, and `imshow` to the matching named window. Toggle flags off and the window is destroyed; toggle on and it springs back. A simple counter warns when more than three windows are open so the user knows why the framerate just dropped.

## Implementation Notes

- **Separable convolutions** for the 5x5 Gaussian and both 3x3 Sobels. Two 1D passes instead of one 2D pass: O(2N) operations per pixel instead of O(N^2).
- **Signed intermediate buffer** (`CV_16SC3`) for the Sobel operators so negative gradients survive before `convertScaleAbs` collapses them back to `uint8`.
- **Cartoonization** is a composition rather than a single filter: blur, quantize, compute gradient magnitude, then black out any pixel whose channel-wise magnitude exceeds a threshold. The threshold and quantization levels are tunable from `vidDisplay.cpp`.
- **Kelvin filter** uses three independent piecewise-linear curves (one per BGR channel) interpolated into a 256-entry LUT and applied with `cv::LUT`. The curve control points come from the Photoshop Kelvin preset.
- **Color transfer** (`colorTransfer.cpp`) follows Reinhard et al.: convert both images to Lab, subtract destination mean, scale by source-to-destination std-dev ratio per channel, add source mean, convert back to BGR.

## Tech

- **Language:** C++ (C++14)
- **Library:** OpenCV 4 (`opencv4` pkg-config)
- **Build:** GNU make + `g++`
- **Capture:** `cv::VideoCapture(0)`, any V4L2 / AVFoundation / DirectShow camera

## Results

Captured at 480p webcam resolution on a 2020 laptop, all filters except the blur-quantize composites run at the native 30 fps capture rate. Snapshots in `Images from Video Stream/` show the cartoonize and Kelvin outputs at their default tunings. `Project Report.pdf` walks through each filter with input/output pairs and runtime observations.

## Author

Dhruvil Parikh - [dparikh79.github.io](https://dparikh79.github.io)

MS Robotics, Northeastern University. Robotics SE II at Globus Medical.

## License

MIT - see [LICENSE](LICENSE).
