### Optical Computation with Resonance-Cavity Projector–Camera Architectures**

This work explores the use of an **optical resonance cavity** in tandem with projector–camera systems for core neural-network operations (e.g., convolution, down/upsampling, and limited fully connected layers). By physically summing or filtering light, we sidestep many of the digital multiply-accumulate (MAC) operations typical of deep learning, offloading them into free space. Although early prototypes used unaligned or unstable hardware, the approach treats the entire projector–cavity–camera pipeline as an empirically learned transform, potentially enabling large-scale photonic computations.

---

### Down/Upsampling Via 2×2 → 4×1 Rasterization

A classical example is **downsampling** from a 2160p projector onto a 1080p camera. Each 2×2 patch of projector pixels maps to a single camera pixel:

1. **Rasterization**: The top 2×2 block of the input image is rearranged (in the projector’s memory) into a 4×1 strip of brightness values.  
2. **Projection**: The projector emits those 4 into the cavity, which the camera sees as a single pixel.  
3. **Physical Averaging**: Intensity accumulates within the resonance chamber and onto that single camera pixel, effectively performing a hardware “sum” (or approximate average).  

This principle generalizes to **upsampling** by inverting the ratio: the projector uses fewer input intensities but spreads them over more physical pixels, letting the camera “expand” each element into a higher-resolution patch.

---

### Convolution Using the Same 4×1 Mapping

The same 2×2 → 4×1 arrangement supports **convolution**:

1. **Patch Extraction**: A sliding 2×2 kernel region (in the original 1080p input) is reshaped into 4 projector pixels.  
2. **Weight Encoding**: Each of these 4 pixels is modulated by serial physical interaction with the "kernel" of the cavity.
3. **Temporal Summation**: The weights are summed across time in the camera pixel, but do not interact directly.
4. **Scan Over the Image**: Repeatedly shift the 2×2 window to generate the next 4×1 projector patch, capturing each convolved output pixel in sequence.

Thus, one can systematically rasterize 2D kernel patches into 4 projector pixels, with the camera’s single-pixel capture acting as the summed convolution result.

---

### Multi-Projector Aggregation

To process higher throughput or accommodate larger feature maps, multiple synchronized projectors can inject parallel data into the same resonance cavity:

- **4 × 1080p Projectors**: Instead of a single 2160p output for downsampling or convolution, four 1080p devices can each handle part of the kernel, merging inside the cavity to form a single, wide-area summation.  
- **Spatial Summation**: From the camera’s perspective, these partial signals appear as a single integrated intensity, effectively “adding” the multiple projector outputs in real time.  Inside the resonance cavity, light bounces multiple times, merging the 4 intensities into one camera pixel that approximates the convolved output for that kernel location.
- **Reduced Per-Projector Complexity**: Each projector can run at standard 1080p settings, simplifying hardware design, while still achieving combined high-resolution summation in free space.

---

### Fully Connected Layers: Scanning vs. Global Refresh

For a small fully connected (FC) layer—say an 8×8 MNIST digit connected to 64 neurons—**each input-neuron weight** needs to multiply the corresponding input pixel:

- **Scanning Projectors**: If each row needs eight projectors (one per neuron across that row), the entire 8×8 → 64 mapping would demand 64 such scanning channels. Each projector sequentially modulates the relevant row pixel intensities. A rolling-shutter camera integrates over time, summing row-by-row contributions.  
  - **Practical Challenge**: 64 projectors is obviously cumbersome, but feasible as a conceptual demonstration of photonic summation.  
- **Single 1080p Projector with Global Refresh**: A micro-LED device that can set all 1080p pixels simultaneously, combined with a global-shutter camera, can handle each neuron’s weighting in parallel. In one frame, each pixel of the projector encodes the product of an input pixel × that neuron’s weight. The camera sees the integrated sum per neuron in a single exposure.  
  - **High Throughput**: A single pass can compute an entire 1080p-scale FC layer.

---

### Frequency Selectivity: The Resonance Cavity

By default, we assume an **optical resonance cavity**—a mirrored enclosure (potentially with curved or “ram’s horn” geometries)—enables repeated internal reflections:

1. **Empirical Transfer Function**: The cavity’s multi-bounce environment can be modeled as a “transfer layer” that merges signals physically. No precise mechanical alignment is strictly required if the system remains stable during operation.  
2. **Spectral Filtering**: Pre-transforming the input via FFT or wavelet (digitally or with additional optical steps) and then feeding it into the cavity allows for partial frequency or wavelet filtering.  
3. **Nonlinear Enhancements**: Curved or partially transmissive media can introduce mild nonlinearities, potentially emulating activation functions (though this remains a more speculative direction).

Because the cavity approach is fully data-driven, slight hardware imperfections or drift can be learned and corrected over time, relying on synchronization, stable reflectivity, and consistent geometry.

---

### Technical Considerations & Future Outlook

1. **Synchronization**  
   - Rolling-shutter cameras pair well with scanning projectors, each line capturing a time-summed intensity.  
   - Global-shutter cameras match global-refresh projectors, capturing the entire weighting pattern at once.  
2. **Hardware Scaling**  
   - Multi-projector or micro-LED-based solutions scale the optical summation concept to large feature maps or wide FC layers, but mechanical constraints (e.g., lens misalignment) remain.  
3. **Reflective Losses**  
   - Each bounce in the cavity attenuates some portion of the signal, requiring brighter sources or more reflective coatings.  
4. **Empirical Machine Learning**  
   - By calibrating projector inputs and camera outputs through end-to-end training, many alignment issues become learning tasks rather than hardware fixes.  
5. **Activation & Higher-Level Blocks**  
   - Nonlinear or more advanced blocks (attention, normalization) often remain in digital software or specialized chips. The resonance-cavity pipeline primarily addresses large linear transforms (convolutions, FC, spectral).

In sum, **projector–camera resonance-cavity computation** offers a photonic alternative for large-scale summation and filtering in deep networks. A single 2160p projector can handle a 2×2 → 4×1 mapping for downsampling or convolution, but multi-projector schemes (e.g., four 1080p units) increase parallel throughput. Fully connected layers range from scanning-based setups with many projectors to a single, global-refresh micro-LED device. Despite current mechanical and optical hurdles, the method promises substantial MAC savings by harnessing free-space parallelism and repeated internal reflections.
