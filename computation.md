**Optical Computation with Projector–Camera Architectures**

This work proposes offloading core neural-network operations—down/upsampling, convolution, and certain fully connected layers—onto an *optical pipeline* driven by a high-resolution projector, synchronized camera, and, optionally, an optical resonance cavity. Although preliminary experiments used unstable prototypes, the results suggest that more robust hardware could perform fundamental linear transforms in free space.

Instead of digital summation, we exploit light-based integration. Rolling-shutter cameras can pair with scanning projectors to integrate intensities *over time*, effectively creating a temporal summation of each kernel patch; in contrast, a *global shutter* camera coupled to a projector with a global refresh (i.e., all pixels update simultaneously) captures the entire weighting pattern at once.

For example, **downsampling** from 2160p to 1080p is achieved by grouping each 2×2 projector-pixel block into one “super-pixel” in the camera’s view, while *inverse mapping* can provide upsampling. Convolution uses the same principle: a 2160p projector encodes 4×4 patches of a 1080p input as weighted intensities, letting the camera sum these contributions into a single convolved pixel.

A hypothetical fully connected (FC) layer for MNIST’s 8×8-digit representation using scanning projectors and rolling shutter cameras would be built via eight micro-projectors per input row, each encoding its neuron’s weights. In contrast, a single *global shutter* camera coupled to a projector with a global refresh (i.e., all pixels update simultaneously) captures the entire weighting pattern at once. 

Likewise, optical resonance cavities can selectively amplify or damp certain frequencies, approximating FFT/DCT or wavelet transforms in one pass. Though hardware stability remains a challenge—especially for resonance-based filtering—such photonic summation drastically reduces digital multiply-accumulate operations. With improved alignment and calibration, this hybrid optical–digital scheme could accelerate CNN inference and spectral analysis by harnessing nature’s own parallel, high-speed integration of light.
