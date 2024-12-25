------

#### **CryptographicCommunication.pseudo**

In this conceptual cryptographic pipeline, **Alice** and **Bob** work together to pass messages securely while **Eve** tries to intercept them. A learned “public key generation” network creates public keys from private keys, so the entire encryption–decryption process is differentiable. Each agent (Alice, Bob, Eve) sees shared environment data—for example, real-time camera captures—to inject additional, ephemeral complexity into the encryption. During training, Alice and Bob jointly minimize Bob’s decryption error while maximizing Eve’s, and Eve attempts to lower her own error. By also optimizing the public key generator to reduce Bob’s loss, the system co-evolves encryption strategies resistant to Eve’s partial observations. While not a real-world cryptographic scheme, this approach shows how learned key production and adversarial objectives can be fused into an end-to-end, trainable secure-communication framework.

**Description:**
Current Multi-Agent systems  with higher levels of reliability contain hard-coded function, but the ultimate goal is to make functional fully-differentiable systems. This toy model implements a stylized cryptographic communication setup in which **Alice** and **Bob** cooperate to exchange messages securely, while **Eve** tries to intercept them. A **PublicKeyGenModel** is also trained to produce public keys given a private key, thus adding an additional trainable component to the system.

```pseudo
BEGIN CryptographicCommunication

    # 1) Initialize agent models
    #    - Alice: encrypts plaintext using a public key, plus environment data
    #    - Bob: decrypts using a private key, tries to recover plaintext
    #    - Eve: attempts eavesdropping with partial data
    AGENT_MODEL_ALICE = InitializeAgentModelAlice()
    AGENT_MODEL_BOB   = InitializeAgentModelBob()
    AGENT_MODEL_EVE   = InitializeAgentModelEve()

    # 2) Initialize public key generation model 
    #    (trainable network that maps PRIVATE_KEY -> PUBLIC_KEY)
    PUBLIC_KEY_GEN_MODEL = InitializePublicKeyGenModel()

    # 3) Initialize optimizers
    OPTIMIZER_ALICE         = InitializeOptimizer(AGENT_MODEL_ALICE.Parameters)
    OPTIMIZER_BOB           = InitializeOptimizer(AGENT_MODEL_BOB.Parameters)
    OPTIMIZER_EVE           = InitializeOptimizer(AGENT_MODEL_EVE.Parameters)
    OPTIMIZER_PUBLIC_KEY_GEN= InitializeOptimizer(PUBLIC_KEY_GEN_MODEL.Parameters)

    # 4) Main training loop (continuous or set number of iterations)
    WHILE True DO

        # (a) Generate new plaintext and private key
        PLAINTEXT   = GeneratePlaintext()
        PRIVATE_KEY = GeneratePrivateKey()

        # (b) PublicKeyGenModel derives a public key from the private key
        PUBLIC_KEY  = PUBLIC_KEY_GEN_MODEL(PRIVATE_KEY)

        # (c) All agents capture some shared environment data (simultaneous images)
        CAMERA_IMAGE_ALICE = CaptureImage()
        CAMERA_IMAGE_BOB   = CaptureImage()
        CAMERA_IMAGE_EVE   = CaptureImage()

        # (d) Each agent produces a projection/encryption signal
        #     - Alice encrypts using PLAINTEXT + PUBLIC_KEY + env image
        #     - Bob tries to decrypt using PRIVATE_KEY + env image
        #     - Eve tries to decrypt with env image alone
        PROJECTION_SIGNAL_ALICE = AGENT_MODEL_ALICE(PLAINTEXT, PUBLIC_KEY, CAMERA_IMAGE_ALICE)

        (PROJECTION_SIGNAL_BOB, DECRYPTED_BOB) =
            AGENT_MODEL_BOB(PRIVATE_KEY, CAMERA_IMAGE_BOB)

        (PROJECTION_SIGNAL_EVE, DECRYPTED_EVE) =
            AGENT_MODEL_EVE(CAMERA_IMAGE_EVE)

        # (e) Project signals: in some conceptual sense, they're broadcast
        ProjectSignal(PROJECTION_SIGNAL_ALICE)
        ProjectSignal(PROJECTION_SIGNAL_BOB)
        ProjectSignal(PROJECTION_SIGNAL_EVE)

        # (f) Compute losses for Bob and Eve w.r.t. original PLAINTEXT
        DECRYPTION_LOSS_BOB = ComputeComparativeLoss(PLAINTEXT, DECRYPTED_BOB)
        DECRYPTION_LOSS_EVE = ComputeComparativeLoss(PLAINTEXT, DECRYPTED_EVE)

        # (g) Each agent's loss
        #     - Alice and Bob want to *minimize Bob's loss* and *maximize Eve's loss*
        #     - Eve tries to *minimize her own loss*
        LOSS_ALICE = DECRYPTION_LOSS_BOB - DECRYPTION_LOSS_EVE
        LOSS_BOB   = DECRYPTION_LOSS_BOB - DECRYPTION_LOSS_EVE
        LOSS_EVE   = DECRYPTION_LOSS_EVE

        # (h) The public key generation model tries to help Bob succeed,
        #     so we tie its loss to Bob's success
        LOSS_PUBLIC_KEY_GEN = DECRYPTION_LOSS_BOB

        # (i) Zero out gradients
        OPTIMIZER_ALICE.ZeroGrad()
        OPTIMIZER_BOB.ZeroGrad()
        OPTIMIZER_EVE.ZeroGrad()
        OPTIMIZER_PUBLIC_KEY_GEN.ZeroGrad()

        # (j) Backprop through each model
        Backpropagate(LOSS_ALICE,         AGENT_MODEL_ALICE)
        Backpropagate(LOSS_BOB,           AGENT_MODEL_BOB)
        Backpropagate(LOSS_EVE,           AGENT_MODEL_EVE)
        Backpropagate(LOSS_PUBLIC_KEY_GEN,PUBLIC_KEY_GEN_MODEL)

        # (k) Update each model
        OPTIMIZER_ALICE.Step()
        OPTIMIZER_BOB.Step()
        OPTIMIZER_EVE.Step()
        OPTIMIZER_PUBLIC_KEY_GEN.Step()

        # (l) Print debugging info
        PRINT("Bob Decryption Loss: ", DECRYPTION_LOSS_BOB)
        PRINT("Eve Decryption Loss: ", DECRYPTION_LOSS_EVE)

    END WHILE

END CryptographicCommunication
```

#### **Brief Explanation**

1. **Agents**
   - **Alice**: Encrypts a plaintext using a (trainable) public key and environment data.
   - **Bob**: Receives some portion of environment data plus a private key, tries to decrypt.
   - **Eve**: Eavesdropper who sees only environment data, tries to decrypt.
2. **PublicKeyGenModel**
   - Learns a mapping from a newly generated private key to a suitable public key.
   - Minimizes Bob’s decryption loss so that Bob can reliably decode the message.
3. **Loss Functions**
   - **Bob** wants low decryption loss (`DECRYPTION_LOSS_BOB`) => correct decoding.
   - **Eve** wants low decryption loss as well, but from Alice & Bob’s viewpoint, we want to maximize Eve’s loss.
   - **Alice** & **Bob** define their losses as BobLoss−EveLoss\text{BobLoss} - \text{EveLoss} so they push Bob’s error down & push Eve’s error up.
   - **Eve** is trained to push her own error down.
   - **PublicKeyGen** also tries to reduce Bob’s loss, effectively learning a “cooperative” key structure.
4. **Environment Data**
   - Agents all share some real-time data (e.g., camera images). This can serve as ephemeral or “one-time pad–like” context.
   - The code simulates them capturing images simultaneously, feeding them as input to their respective models.
5. **Adversarial Interaction**
   - Over many iterations, the system evolves so that **Alice & Bob** find encryption strategies that remain robust to Eve’s attempts.
   - Eve tries to break the encryption only from partial info.
   - The **PublicKeyGenModel** is a novel twist where even the public key’s formation is learned.

------

#### **Key Takeaways**

- **Trainable Key Generation**: By making the public key also produced by a neural network, the entire cryptosystem can co-evolve in a differentiable way.
- **Adversarial Learning**: Agents have competing objectives—(Alice,Bob) vs. Eve—implemented by their respective losses.
- **Environmental Data**: The camera images add ephemeral secrecy, making it harder for Eve to replicate or predict signals exactly.

This script is of course conceptual and **not** guaranteed secure in real-world cryptographic sense, but it serves to illustrate how one might set up a machine-learning-driven cryptographic ecosystem in a pseudo-coded environment.
