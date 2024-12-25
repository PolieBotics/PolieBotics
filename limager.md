**Limager: Emission–Recording Pairs for NeRF-Based Scene Reconstruction**  
We present a prototype “Limager” system that extends Neural Radiance Fields (NeRF) to handle projector–camera loops, jointly optimizing camera calibration and scene structure from unsupervised image pairs. Unlike standard NeRF pipelines, which rely on passive imagery, the Limager includes an *emission* component—light patterns actively projected into the environment—alongside the captured images. These emission–recording pairs drive an end-to-end training objective in which the model infers both scene geometry and appearance. 

In practice, *Limager* automatically adjusts camera intrinsics and extrinsics throughout training, aided by an onboard Inertial Measurement Unit (IMU) that can provide coarse prior estimates of pose. While current hardware is tested only in small-scale indoor settings with limited stability, the underlying approach generalizes readily to larger scenes and more advanced machine learning methods (e.g., semantic labeling, instance segmentation). The essential pipeline is:

1. **Data Acquisition**  
   *Emission signals* (light patterns) are projected into the scene; a camera captures each *recording*. This forms a set of \((\text{emission}, \text{recording})\) pairs, implicitly capturing scene geometry, reflectance, and the projector’s influence on intensity and color.

2. **Camera and NeRF Initialization**  
   The system initializes a NeRF model (parameterized by a small neural network) plus camera parameters (intrinsics/extrinsics for each frame). The IMU data helps seed camera pose estimates but is optional.

3. **Joint Training**  
   For each emission–recording pair, the method:
   - **Generates Rays:** Using the camera intrinsics/extrinsics, it constructs pixel-wise rays.  
   - **NeRF Rendering:** The model predicts densities and colors per sampled ray, incorporating the emission features to account for projector-induced illumination.  
   - **Loss Computation:** The rendered output is compared to the real captured image (e.g., mean squared error).  
   - **Optimization:** The network and camera parameters are updated via gradient-based methods.  

4. **Self-Calibration & Reconstruction**  
   Over many iterations, the camera extrinsics converge to align with the IMU or other pose constraints, while the NeRF’s weights capture the scene’s spatial structure and reflectance properties.  

5. **Evaluation**  
   A typical metric is PSNR, measured between the rendered scene from the learned model and the actual camera captures. The final model can synthesize novel viewpoints, or re-render the scene under different emission patterns.

Although our *Limager* prototype remains limited in scale, it illustrates how *any* machine learning or classification approach can be interwoven with projector–camera loops. Emission signals provide additional control and supervision—potentially enabling faster convergence, improved geometry, or domain-specific tasks (e.g., 3D labeling, defect detection). By wrapping NeRF or other neural pipelines in a physically interactive setup, we move toward richer, self-calibrating systems for real-world scene understanding.

------

#### **UnsupervisedNeRFTraining.pseudo**

**Description:**
Trains a Neural Radiance Field (NeRF) model to reconstruct scene geometry and appearance given **emission–recording** pairs. In theory, the camera intrinsics and extrinsics are also jointly optimized, so the system self-calibrates from the data. The current versions hardware has an Inertial Measurement Unit for use in assisting training, and has only been tested at small scales.

```pseudo
BEGIN UnsupervisedNeRFTraining

    # 1) Load the emission-recording pairs dataset
    DATASET_DIR = "path/to/data"
    EMISSION_RECORDING_PAIRS = LoadEmissionRecordingPairs(DATASET_DIR)
    DATASET_SIZE = Length(EMISSION_RECORDING_PAIRS)

    # 2) Initialize camera parameters
    (IMAGE_WIDTH, IMAGE_HEIGHT) = GetImageDimensions(EMISSION_RECORDING_PAIRS)
    FOCAL_LENGTH   = InitializeFocalLength()
    CAMERA_INTRINSICS = BuildCameraIntrinsics(FOCAL_LENGTH, IMAGE_WIDTH, IMAGE_HEIGHT)
    CAMERA_EXTRINSICS = InitializeCameraExtrinsics(DATASET_SIZE)
       # e.g. each item might be a 4×4 transform

    # 3) Initialize NeRF model
    INPUT_CHANNELS   = 3 + EMISSION_DIM  # e.g. (x,y,z) + emission features
    OUTPUT_CHANNELS  = 4                # (R,G,B) + density
    NUM_LAYERS       = 8
    HIDDEN_UNITS     = 256
    NeRF_Model = InitializeNeRFModel(
                     INPUT_CHANNELS,
                     OUTPUT_CHANNELS,
                     NUM_LAYERS,
                     HIDDEN_UNITS
                 )

    # 4) Initialize optimizers for NeRF + camera parameters
    LEARNING_RATE = 0.0005
    Optim_NeRF       = InitializeAdam(NeRF_Model.Parameters, LEARNING_RATE)
    Optim_Intrinsics = InitializeAdam(CAMERA_INTRINSICS,     LEARNING_RATE)
    Optim_Extrinsics = InitializeAdam(CAMERA_EXTRINSICS,     LEARNING_RATE)

    # 5) Main training hyperparameters
    NUM_EPOCHS           = 100
    ITERATIONS_PER_EPOCH = 1000

    # 6) Training loop
    FOR epoch FROM 1 TO NUM_EPOCHS DO
        EPOCH_LOSS = 0

        FOR iter FROM 1 TO ITERATIONS_PER_EPOCH DO
            # (a) Sample a random data point
            i = RandomInteger(0, DATASET_SIZE - 1)
            (EMISSION, RECORDING) = EMISSION_RECORDING_PAIRS[i]

            # (b) Retrieve camera parameters (intrinsic + extrinsic)
            CurrentIntr  = CAMERA_INTRINSICS
            CurrentExtr  = CAMERA_EXTRINSICS[i]

            # (c) Generate rays for each pixel
            (RaysOrig, RaysDir) = GenerateRays(CurrentIntr, CurrentExtr,
                                               IMAGE_WIDTH, IMAGE_HEIGHT)

            # (d) Render image from NeRF
            RenderedImg = RenderImageFromNeRF(NeRF_Model,
                                              RaysOrig, RaysDir,
                                              EMISSION)

            # (e) Compare with actual recording
            Loss = MeanSquaredError(RenderedImg, RECORDING)
            EPOCH_LOSS = EPOCH_LOSS + Loss

            # (f) Backprop + update
            Optim_NeRF.ZeroGrad()
            Optim_Intrinsics.ZeroGrad()
            Optim_Extrinsics.ZeroGrad()

            Backprop(Loss)
            Optim_NeRF.Step()
            Optim_Intrinsics.Step()
            Optim_Extrinsics.Step()

        END FOR

        # (g) Average epoch loss
        AVG_LOSS = EPOCH_LOSS / ITERATIONS_PER_EPOCH
        Print("Epoch=", epoch, "AvgLoss=", AVG_LOSS)

    END FOR

    # 7) Verification / evaluation phase
    #    e.g. measure PSNR for each sample
    PSNR_SCORES = EmptyList()

    FOR i FROM 0 TO (DATASET_SIZE - 1) DO
        (EMIT, RECS) = EMISSION_RECORDING_PAIRS[i]
        Intr  = CAMERA_INTRINSICS
        Extr  = CAMERA_EXTRINSICS[i]

        (RaysOrig, RaysDir) = GenerateRays(Intr, Extr, IMAGE_WIDTH, IMAGE_HEIGHT)
        PredImg = RenderImageFromNeRF(NeRF_Model, RaysOrig, RaysDir, EMIT)

        psnr_val = ComputePSNR(PredImg, RECS)
        Append(PSNR_SCORES, psnr_val)
        Print("Sample=", i, "PSNR=", psnr_val)
    END FOR

    # 8) Summary metric
    OverallPSNR = Average(PSNR_SCORES)
    Print("Overall PSNR:", OverallPSNR)

END UnsupervisedNeRFTraining
```

------

#### **Support Functions**

Below are representative placeholders for the subroutines invoked in the main pseudo-code. Adapt them as needed for your actual pipeline.

```pseudo
FUNCTION GenerateRays(Intrinsic, Extrinsic, Width, Height)
    # 1) Compute pixel coordinates
    PixelCoords = CreatePixelGrid(Width, Height)

    # 2) Convert to camera coordinates (using Intrinsic)
    CameraCoords = Inverse(Intrinsic) * PixelCoords

    # 3) Transform to world coordinates (using Extrinsic)
    RaysDir     = Normalize(Extrinsic.Rotation * CameraCoords)
    RaysOrigin  = ExtractCameraPosition(Extrinsic) # e.g. -R^T * T

    RETURN (RaysOrigin, RaysDir)
END FUNCTION

FUNCTION RenderImageFromNeRF(NeRFModel, RaysOrigin, RaysDirection, Emission)
    # 1) For each ray => sample points along ray
    # 2) Query NeRF => get color + density
    # 3) Volume render => accumulate final pixel color
    # 4) Collect all pixels => form rendered image
    RenderedImg = RayMarchingLoop(NeRFModel, RaysOrigin, RaysDirection, Emission)
    RETURN RenderedImg
END FUNCTION

FUNCTION MeanSquaredError(Prediction, GroundTruth)
    # Standard MSE over image pixels
    RETURN Sum((Prediction - GroundTruth)^2) / NumPixels
END FUNCTION

FUNCTION ComputePSNR(PredImg, TrueImg)
    mse = MeanSquaredError(PredImg, TrueImg)
    if mse <= 1e-10 then mse = 1e-10
    psnr_val = 10 * log10(1 / mse)  # assuming images are normalized [0..1]
    RETURN psnr_val
END FUNCTION
```

------

#### **Key Steps Recap**

1. **Load** emission-recording pairs and set up camera + NeRF model.
2. **Train** by sampling random pairs, generating rays, rendering with NeRF, computing MSE, and updating model/camera parameters.
3. **Verify** the trained model by rendering each sample, measuring PSNR, and computing an overall accuracy/quality metric.

This pseudo-code demonstrates the core concepts of an **unsupervised** approach: the system sees only `(emission, recording)` pairs and automatically **(1)** calibrates camera extrinsics/intrinsics, **(2)** learns scene geometry and appearance via NeRF, and **(3)** incorporates the emission signal as an additional model input.
