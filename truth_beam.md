**Abstract**  
We present the *Truth Beam*, a proof-of-concept pipeline that combines cryptographic hash chaining with a neural forensic model for projector–camera recording authenticity. By anchoring Perlin noise seeds to a public blockchain, the *Truth Beam* assigns each projected frame a verifiable cryptographic fingerprint. A straightforward hashing scheme ensures sequential frame integrity, while a small autoencoder–GAN component detects emission–capture anomalies. *Truth Beam* is not unbreakable, but it demonstrates how blockchain-sourced randomness and machine-learning–based forensics can intersect to increase transparency and pave the way for more sophisticated video integrity mechanisms in future forensic and AI endeavors.

**Description**  
The *Truth Beam* process starts by retrieving a fresh blockhash from a public blockchain—this blockhash seeds a Perlin noise generator, whose output is projected onto a scene. A co-located camera records these emissions in real time, and each captured frame is hashed in sequence so that any tampering breaks the chain. The final or intermediate hashes can be posted back to the blockchain for external proof of continuity.

Alongside the hash chaining, a simple autoencoder–GAN is trained to reconstruct paired “emission–capture” data, with signficant deviations from the originals being indiciative of forgery or adulteration, and discriminator trained on the latent space representation of these data estimates their veracity. While the *Truth Beam* does not claim foolproof security, it showcases how cryptographically anchored noise and learned forensics could work together in practice. We encourage further exploration and scaling of this concept to stimulate fresh approaches and adversarial evolution in both cryptographic and ML-based video verification.

------

#### 1) **SecureRecord.pseudo**

**Description:**
 Captures frames from a camera while projecting structured noise (e.g., Perlin) seeded by a cryptographic hash chain. Saves emissions (noise), recordings (camera frames), and the evolving hash chain.

```pseudo
BEGIN SecureRecording

    # Obtain a fresh RSK blockhash to seed the process
    RSK_BLOCKHASH = GetFreshBlockhashFromRSK()
    INIT_VEC      = HashFunction(RSK_BLOCKHASH)

    # Create a directory named after the initialization vector (or blockhash)
    DIRECTORY_NAME = "recordings_" + TruncateOrEncode(INIT_VEC)
    CreateDirectory(DIRECTORY_NAME)

    # Define how many frames to record
    LOOP_COUNT = 100  # Example

    # Initialize data containers in memory
    EMISSIONS_LIST   = EmptyList()
    RECORDINGS_LIST  = EmptyList()
    HASHES_LIST      = EmptyList()

    # Current hash to chain with each new capture
    CURRENT_HASH = INIT_VEC

    # Start pipeline (Camera + optional projector)
    InitializeHardwareAndPipeline()

    FOR i FROM 1 TO LOOP_COUNT DO:

        # 1. Generate Perlin-based noise or structured emission from CURRENT_HASH
        STRUCTURED_NOISE = StructuredNoiseGenerator(CURRENT_HASH)

        # 2. Display or project the structured noise
        EmitProjectionSignal(STRUCTURED_NOISE)

        # 3. Capture the camera image that includes the projected pattern
        CAPTURED_IMAGE = CaptureImage()

        # 4. Append relevant data to local lists for later saving
        Append(EMISSIONS_LIST, STRUCTURED_NOISE)
        Append(RECORDINGS_LIST, CAPTURED_IMAGE)
        Append(HASHES_LIST, CURRENT_HASH)

        # 5. Update the CURRENT_HASH to link it with the newly captured frame
        CURRENT_HASH = HashFunction(CURRENT_HASH, CAPTURED_IMAGE)

    END FOR

    # Optionally submit the final hash to RSK or another blockchain
    SubmitHashToRSK(CURRENT_HASH)   # or skip if dummy run

    # Save the final data into the directory
    SaveToDirectory(DIRECTORY_NAME, EMISSIONS_LIST, "emissions")
    SaveToDirectory(DIRECTORY_NAME, RECORDINGS_LIST, "recordings")
    SaveToDirectory(DIRECTORY_NAME, HASHES_LIST, "hashes")

    # Close and clean up hardware/pipeline
    StopHardwareAndPipeline()

END SecureRecording
```

------

#### 2) **VerifyHashesAndPerlinMSE.pseudo**

**Description:**
 Loads a previously recorded dataset, verifies the cryptographic hash chain, and re-generates Perlin noise to compare MSE with the stored emission. This ensures no tampering in the local chain and checks whether the recorded structured noise matches re-generation.

```pseudo
BEGIN VerifyCryptoAndPerlinMSE

    # 1) Load the recorded dataset
    DATASET_DIR = "recordings_someIV"  # example folder name
    EMISSIONS   = LoadImages(DATASET_DIR, "emissions")
    RECORDINGS  = LoadImages(DATASET_DIR, "recordings")
    HASHES      = LoadHashes(DATASET_DIR, "hashes")

    N = Length(EMISSIONS)  # should match RECORDINGS, HASHES, etc.

    # 2) Retrieve initial hash (some attribute or first entry)
    INITIAL_HASH = SomeMethodToGetInitialHash(DATASET_DIR)

    # 3) Iterate frames:
    current_hash = INITIAL_HASH
    RESULTS = EmptyList()

    FOR i FROM 0 TO (N-1) DO

        # (a) Check local cryptographic chain
        expected_next = HashFunction(current_hash, RECORDINGS[i])
        actual_stored = HASHES[i]
        hash_ok       = (expected_next == actual_stored)

        # (b) Re-generate Perlin from prior hash => compare MSE
        re_generated = GeneratePerlinFromHash(current_hash)
        mse_val      = ComputeMSE(re_generated, EMISSIONS[i])

        # Store results
        Append(RESULTS, {
            "index"     : i,
            "hash_ok"   : hash_ok,
            "mse_perlin": mse_val
        })

        # Update current_hash
        current_hash = actual_stored

    END FOR

    # 4) Print final results
    Print("=== Verification Results: Hash + Perlin MSE ===")
    FOREACH item IN RESULTS DO
        Print("Frame=", item.index,
              " HashOK=", item.hash_ok,
              " PerlinMSE=", item.mse_perlin)
    END FOREACH

END VerifyCryptoAndPerlinMSE
```

------

#### 3) **TrainAEandGAN.pseudo**

**Description:**
 Trains a simple conceptual Adversarial Autoencoder (AE) to combine or reconstruct `(emission, recording)` pairs, and then trains a Latent-GAN on the encoder’s latent space.

```pseudo
BEGIN TrainAEandGAN

    # 1) Load dataset of (emission, recording) pairs
    DATASET_DIR = "training_data"
    DATA = LoadDatasetPairs(DATASET_DIR)  # list of (emission, recording)

    # 2) Initialize an Adversarial Autoencoder model
    AAE_MODEL = InitializeAAEModel()

    # 3) Train AAE
    EPOCHS_AAE = 10
    FOR epoch FROM 1 TO EPOCHS_AAE DO
        FOREACH (emi_batch, rec_batch) in DATA DO
            # Encode
            latents = AAE_MODEL.Encoder(emi_batch, rec_batch)
            # Decode
            recons  = AAE_MODEL.Decoder(latents)
            
            # measure reconstruction
            loss_recon = ReconstructionLoss(recons, emi_batch, rec_batch)
            loss_adv   = AAE_MODEL.ComputeAdversarialLoss(...)
            total_loss = loss_recon + loss_adv

            AAE_MODEL.Backpropagate(total_loss)
        END FOREACH
        Print("Epoch", epoch, "done (AAE).")
    END FOR

    # 4) Initialize LatentGAN (a generator + discriminator on latent space)
    LATENT_GAN = InitializeLatentGAN()

    # 5) Train LatentGAN
    EPOCHS_GAN = 5
    FOR epoch FROM 1 TO EPOCHS_GAN DO
        FOREACH (emi_batch, rec_batch) in DATA DO
            # get real latents
            real_latents = AAE_MODEL.Encoder(emi_batch, rec_batch)
            # generate fake latents from random noise
            noise        = SampleNoise(SizeOf(real_latents))
            fake_latents = LATENT_GAN.Generator(noise)

            # update Discriminator
            disc_loss = LATENT_GAN.UpdateDiscriminator(real_latents, fake_latents)
            # update Generator
            gen_loss  = LATENT_GAN.UpdateGenerator()
        END FOREACH
        Print("Epoch", epoch, "done (LatentGAN).")
    END FOR

    # 6) Save models
    SaveModel(AAE_MODEL, "aae_model.dat")
    SaveModel(LATENT_GAN, "latent_gan.dat")

END TrainAEandGAN
```

------

#### 4) **VerifyUsingAEandGAN.pseudo**

**Description:**
 Loads the trained AAE + LatentGAN. For each new sample `(emission, recording)`, it checks reconstruction error and a latent authenticity score from the GAN’s discriminator.

```pseudo
BEGIN VerifyUsingAEandGAN

    # 1) Load final models
    AAE_MODEL   = LoadModel("aae_model.dat")
    LATENT_GAN  = LoadModel("latent_gan.dat")

    # 2) Load novel pairs to verify
    DATASET_DIR = "verification_data"
    VERIFY_DATA = LoadDatasetPairs(DATASET_DIR)

    # 3) Prepare a results log
    RESULTS_LOG = EmptyList()

    # 4) For each pair, do AE recon + latent disc check
    FOREACH index, (emi, rec) in VERIFY_DATA DO
        
        # AE: encode + decode
        lat_z         = AAE_MODEL.Encoder(emi, rec)
        recon_e, recon_r = AAE_MODEL.Decoder(lat_z)
        recon_error   = L1orMSE((recon_e, recon_r), (emi, rec))

        # Latent authenticity
        disc_value = LATENT_GAN.Discriminator(lat_z)
        
        # record results
        Append(RESULTS_LOG, {
            "index"      : index,
            "recon_err"  : recon_error,
            "disc_score" : disc_value
        })

    END FOREACH

    # 5) Print or save final verification results
    Print("=== AE & Latent GAN Verification ===")
    FOREACH item in RESULTS_LOG DO
        Print("Index=", item.index,
              "ReconErr=", item.recon_err,
              "DiscScore=", item.disc_score)
    END FOREACH

END VerifyUsingAEandGAN
```

------

#### Final Notes

1. **`SecureRecord.pseudo`** ensures every captured frame is hashed into a chain, and the Perlin noise (emission) is saved.
2. **`VerifyHashesAndPerlinMSE.pseudo`** reconstructs the emission from the previous hash to check authenticity of both chain and noise.
3. **`TrainAEandGAN.pseudo`** is a simplified conceptual script to train an autoencoder (AAE) and then a latent GAN on (emission, recording) data.
4. **`VerifyUsingAEandGAN.pseudo`** loads those trained models and checks new samples’ reconstruction error and authenticity.
