------

#### High-Level Explanation of The Training of an Orb Reactor for Truth Beam Verification

This is a two-input orb reactor, with the emissions and recordings corresponding to a Truth Beam recording being processed using a PoliePuter with two projector inputs and one camera output.

1. **Physical Emitter/Recorder as RL Agents**
   - Because the physical projection process involves real hardware (the “chamber” plus a camera), it’s hard to backpropagate gradients directly through the environment.
   - Instead, we model each of these networks—**EmissionNet** (for emission signals) and **RecordingNet** (for recording signals)—as **RL agents** that choose how to process or transform the signals for each “episode.”
   - Their reward: produce outputs that, once projected and captured, the **Discriminator** deems “authentic.”
2. **Generator + Discriminator in Classic Adversarial Training**
   - We still have a typical generator GG (the “Faker” or “Adversarial Generator”) that tries to produce a **fake camera image** to fool DD.
   - The **Discriminator** DD classifies camera captures as real or fake.
   - GG and DD can be trained using **standard gradient-based backprop** (since they operate purely in the digital domain: generating or classifying images).
3. **Combined Setup**
   - In each iteration (“episode”):
     1. **EmissionNet** and **RecordingNet** produce “actions” (processed signals) to be projected.
     2. The environment physically projects those signals into the chamber, capturing a real camera image (REAL_IMAGE\text{REAL\_IMAGE}).
     3. The adversarial generator GG (digital-only) also produces a **fake** camera image (FAKE_IMAGE\text{FAKE\_IMAGE}).
     4. The discriminator DD tries to classify both REAL_IMAGE\text{REAL\_IMAGE} and FAKE_IMAGE\text{FAKE\_IMAGE}.
     5. Rewards for the RL agents (emission/recording) come from the discriminator labeling REAL_IMAGE\text{REAL\_IMAGE} as real.
     6. GG and DD are updated using normal GAN losses (i.e., FAKE_IMAGE\text{FAKE\_IMAGE} labeled real for GG’s success, real labeled real for DD’s success, etc.).
     7. The RL agents also update their policies (e.g., via policy gradients, Q-learning, or any RL algorithm) based on the reward signals.

------

## **Pseudocode**: Mixed RL + Adversarial

```pseudo
BEGIN MixedReactorTraining

    # 1) Initialize environment (chamber + camera)
    ENV = InitializePhysicalEnvironment()

    # 2) RL Agents for physical emission/recording
    RL_EMISSION_AGENT   = InitializeRLAgent(policy=EmissionPolicy)
    RL_RECORDING_AGENT  = InitializeRLAgent(policy=RecordingPolicy)

    # 3) Classic Generator + Discriminator
    G = InitializeGenerator()      # Adversarial "fake camera image" generator
    D = InitializeDiscriminator()  # Classifier for real vs. fake camera images

    # 4) Load dataset of real emission/recording pairs
    REAL_PAIRS = LoadRealEmissionRecordingPairs()  # e.g. (EmissionInput, RecordingInput)

    # 5) Training loop: episodes + mini-batch of real pairs
    MAX_EPISODES = 1000
    FOR episode IN 1..MAX_EPISODES DO

        Shuffle(REAL_PAIRS)
        FOR each PAIR in REAL_PAIRS DO
            # (A) RL agents select actions
            #     Based on environment or memory
            ACTION_EMISSION  = RL_EMISSION_AGENT.SelectAction(ENV.State)
            ACTION_RECORDING = RL_RECORDING_AGENT.SelectAction(ENV.State)

            # (B) Environment step -> actual projection + capture
            #     Emission & Recording signals are physically projected into the chamber
            #     The camera produces REAL_IMAGE
            REAL_IMAGE = ENV.PhysicalStep(ACTION_EMISSION, ACTION_RECORDING)

            # (C) Generator produces FAKE_IMAGE
            FAKE_IMAGE = G(NoiseInput())

            # (D) Discriminator outputs
            D_REAL = D(REAL_IMAGE)
            D_FAKE = D(FAKE_IMAGE)

            # (E) Rewards for RL agents
            #     If the discriminator is more confident that REAL_IMAGE is real -> higher reward
            #     If the discriminator is uncertain or lower => lower reward
            #     (Implementation detail depends on RL design)
            REWARD_EMISSION, REWARD_RECORDING = ComputeRLRewards(D_REAL)

            # (F) RL agents update from transitions
            RL_EMISSION_AGENT.UpdatePolicy(ENV.State, ACTION_EMISSION, REWARD_EMISSION, NextState(ENV))
            RL_RECORDING_AGENT.UpdatePolicy(ENV.State, ACTION_RECORDING, REWARD_RECORDING, NextState(ENV))

            # (G) Classic adversarial updates for G + D
            #     1) Discriminator loss
            DISC_LOSS = LossReal(D_REAL) + LossFake(D_FAKE)
            D.Optimize(DISC_LOSS)

            #     2) Generator wants D_FAKE -> real
            GEN_LOSS = LossGenerator(D_FAKE -> realLabel)
            G.Optimize(GEN_LOSS)

        END FOR

        # Possibly track or log average RL reward & adversarial losses
        Print("Episode ", episode, " done.")

    END FOR

END MixedReactorTraining
```

### Key Points

1. **Environment Step**
   - PhysicalStep()\text{PhysicalStep()} literally uses the RL agents’ chosen signals for emission/recording, projects them, and records the outcome with the real camera.
   - No direct gradient is passed through the environment—**the RL approach** updates the emission/recording networks from reward signals alone.
2. **Adversarial Update**
   - The generator GG and discriminator DD do standard backprop (digital domain only).
   - GG tries to produce FAKE_IMAGE\text{FAKE\_IMAGE} that the discriminator classifies as real.
   - DD tries to separate real from fake images accurately.
3. **Reinforcement Learning Agents**
   - The **emission** and **recording** processes become RL agents because the real environment is non-differentiable. They each choose an “action” for how to transform or process the raw data.
   - Their reward: the real image is recognized as real by DD.
4. **Possible Variation**
   - Could add negative rewards for the RL agents if the generator FAKE_IMAGE\text{FAKE\_IMAGE} is deemed real. This would further push them to produce “distinctly real” signals that differ from any plausible fake.

------

### Why This Helps

- **Physical Gap**: The emission/recording step is a black box, so we cannot do standard backprop.
- **Adversarial**: We still want a typical G+D dynamic, which is straightforward in the digital domain.
- **RL for E+R**: Emission/recording networks each “learn” how to manipulate or adapt signals in ways the environment (chamber + camera) will produce images that fool or satisfy the discriminator’s real classification, all by receiving scalar reward feedback from the environment outcome.

Thus, you end up with a *mixed training scheme*:

- **Reinforcement** for the real-world side (emission/recording)
- **Gradient-based** for the purely digital generator/discriminator.
