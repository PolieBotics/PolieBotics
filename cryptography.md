
---

### **CryptographicCommunication.pseudo**

**Conceptual Overview**  
Alice, Bob, and Eve share real-time environment data (e.g. images) for ephemeral encryption. **Alice** encrypts a plaintext using a trainable **PublicKeyGenModel** that maps private keys to public keys, while **Bob** attempts to decrypt from another environment snapshot, and **Eve** eavesdrops with only partial data. Alice and Bob jointly minimize Bob’s decryption loss and maximize Eve’s, while Eve tries to minimize her own loss.

```pseudo
BEGIN CryptographicCommunication

    # 1) Initialize agent models
    #    Alice: encrypts plaintext (with Public Key + environment)
    #    Bob:   decrypts with Private Key + environment
    #    Eve:   attempts decryption from environment only
    AGENT_MODEL_ALICE = InitializeAgentModelAlice()
    AGENT_MODEL_BOB   = InitializeAgentModelBob()
    AGENT_MODEL_EVE   = InitializeAgentModelEve()

    # 2) Trainable map: PRIVATE_KEY -> PUBLIC_KEY
    PUBLIC_KEY_GEN_MODEL = InitializePublicKeyGenModel()

    # 3) Create optimizers
    OPTIMIZER_ALICE         = InitializeOptimizer(AGENT_MODEL_ALICE.Parameters)
    OPTIMIZER_BOB           = InitializeOptimizer(AGENT_MODEL_BOB.Parameters)
    OPTIMIZER_EVE           = InitializeOptimizer(AGENT_MODEL_EVE.Parameters)
    OPTIMIZER_PUBLIC_KEY_GEN= InitializeOptimizer(PUBLIC_KEY_GEN_MODEL.Parameters)

    # 4) Main training loop
    WHILE True DO

        # (a) Generate plaintext and private key
        PLAINTEXT   = GeneratePlaintext()
        PRIVATE_KEY = GeneratePrivateKey()

        # (b) From PRIVATE_KEY, compute a PUBLIC_KEY
        PUBLIC_KEY = PUBLIC_KEY_GEN_MODEL(PRIVATE_KEY)

        # (c) Capture environment images for initial encryption step
        CAM_IMG_ALICE = CaptureImage()
        CAM_IMG_BOB   = CaptureImage()
        CAM_IMG_EVE   = CaptureImage()

        # (d) Agents produce encryption signals
        #     Alice uses PLAINTEXT+PUBLIC_KEY+env
        SIG_ALICE = AGENT_MODEL_ALICE(PLAINTEXT, PUBLIC_KEY, CAM_IMG_ALICE)

        #     Bob/Eve initially produce their own signals (e.g. placeholders)
        SIG_BOB = AGENT_MODEL_BOB(PRIVATE_KEY, CAM_IMG_BOB)  # output only
        SIG_EVE = AGENT_MODEL_EVE(CAM_IMG_EVE)               # output only

        # (e) Project each signal into the environment
        ProjectSignal(SIG_ALICE)
        ProjectSignal(SIG_BOB)
        ProjectSignal(SIG_EVE)

        # (f) Capture second-round environment images for actual decryption
        CAM_IMG_ALICE_2 = CaptureImage()
        CAM_IMG_BOB_2   = CaptureImage()
        CAM_IMG_EVE_2   = CaptureImage()

        # (g) Bob/Eve now attempt final decryption from new images
        DECRYPTED_BOB = AGENT_MODEL_BOB(PRIVATE_KEY, CAM_IMG_BOB_2)
        DECRYPTED_EVE = AGENT_MODEL_EVE(CAM_IMG_EVE_2)

        # (h) Compute decryption losses vs. the original PLAINTEXT
        LOSS_BOB_DECR = ComputeComparativeLoss(PLAINTEXT, DECRYPTED_BOB)
        LOSS_EVE_DECR = ComputeComparativeLoss(PLAINTEXT, DECRYPTED_EVE)

        # (i) Agents' losses
        #     Alice+Bob => minimize Bob’s loss & maximize Eve’s
        #     Eve => minimize her own loss
        LOSS_ALICE = LOSS_BOB_DECR - LOSS_EVE_DECR
        LOSS_BOB   = LOSS_BOB_DECR - LOSS_EVE_DECR
        LOSS_EVE   = LOSS_EVE_DECR

        # (j) Public key generator: aims to help Bob
        LOSS_PUBLIC_KEY_GEN = LOSS_BOB_DECR

        # (k) Zero gradients
        OPTIMIZER_ALICE.ZeroGrad()
        OPTIMIZER_BOB.ZeroGrad()
        OPTIMIZER_EVE.ZeroGrad()
        OPTIMIZER_PUBLIC_KEY_GEN.ZeroGrad()

        # (l) Backprop in each model
        Backpropagate(LOSS_ALICE,         AGENT_MODEL_ALICE)
        Backpropagate(LOSS_BOB,           AGENT_MODEL_BOB)
        Backpropagate(LOSS_EVE,           AGENT_MODEL_EVE)
        Backpropagate(LOSS_PUBLIC_KEY_GEN,PUBLIC_KEY_GEN_MODEL)

        # (m) Optimizer steps
        OPTIMIZER_ALICE.Step()
        OPTIMIZER_BOB.Step()
        OPTIMIZER_EVE.Step()
        OPTIMIZER_PUBLIC_KEY_GEN.Step()

        # (n) Debug logs
        PRINT("Bob Decryption Loss:", LOSS_BOB_DECR)
        PRINT("Eve Decryption Loss:", LOSS_EVE_DECR)

    END WHILE

END CryptographicCommunication
```

---

### **Key Points**  
1. **Two-Round Image Capture**  
   - First capture: used by Alice to encrypt.  
   - Second capture: used by Bob/Eve to decrypt.  
   This separation injects ephemeral scene changes between encryption and decryption.

2. **Loss Functions**  
   - **Bob** and **Alice** minimize Bob’s final decryption loss while maximizing Eve’s loss.  
   - **Eve** seeks to minimize her own decryption loss.

3. **Public Key Generation**  
   - A network (`PUBLIC_KEY_GEN_MODEL`) learns how to produce *cooperative* keys that benefit Bob’s decryption specifically.

4. **Practical Limits**  
   - The environment data acts like a one-time pad—somewhat ephemeral. Real security remains purely illustrative; the system is for conceptual, differentiable cryptographic modeling.

Overall, this pseudo-code highlights a *two-phase capture* structure enabling Bob and Eve to reacquire the scene for final decryption—further emphasizing how ephemeral context complicates eavesdropping attempts.
