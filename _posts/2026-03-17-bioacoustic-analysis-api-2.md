---
title: "Bioacoustic analysis API for Orcasound hydrophones (part 2) -- the UX story: why this matters"
date: 2026-03-17
---

The strategy is not to throw away Orcasound's existing spectrogram workflow, but to add a stronger data layer beneath it. The current `orcasite` bouts viewer uses a GraphQL mutation to trigger AWS Lambda jobs that turn raw audio segments into pre-rendered PNG spectrograms. That was a good first step because it got spectrograms into the product quickly. The `ambient-sound-analysis-api` moves the platform toward a more flexible foundation: it serves structured PSD data from parquet files already stored in S3, so the frontend can render spectrograms directly from numeric time-frequency data instead of only displaying flattened images. In practice, that means the PNG flow can remain useful for fixed previews and moderation workflows, while the PSD/parquet flow opens the door to a more interactive analysis experience.

Why this matters is that it shifts Orcasound from static audio images toward exploratory acoustic analysis. A PSD-backed interface can support browser-side rendering modes like dynamic color remapping, threshold highlighting, band emphasis, anomaly views, and masked-versus-unmasked comparisons without regenerating backend image assets. It also gives a better technical foundation for WebGL, where zooming can mean re-rendering the actual visible data region instead of stretching a bitmap. That does not automatically guarantee conservation outcomes, but it does make the acoustic environment more legible. For Orcasound, that is the point: not just detecting whether orcas are present, but building the tools to study vessel noise, masking risk, and the relationship between sound and whale behavior in a way that can eventually support stronger scientific interpretation and better management decisions.

- spectrogram heatmap
- band emphasis
- threshold-highlighted heatmap
- anomaly view
- masked/unmasked comparison 
- change palette, contrast, dB clipping instantly
- WebGL shader can remap values live
- zoom can mean rerendering the actual data region at the - current viewport
- better fidelity
- better interpolation choices

User stories -- 
As a XXX I want to ask questions of the dataset being captured, so can XXX.

#### Core research user
As a bioacoustics researcher, I want to ask questions of the hydrophone dataset being captured, so I can identify time-frequency patterns, test hypotheses, and interpret how vessel noise and other acoustic events may affect orca habitat.

#### Conservation/management user
As a conservation analyst, I want to ask questions of the hydrophone dataset being captured, so I can understand when and where noise conditions may contribute to masking risk and support evidence-based mitigation decisions.

#### Data science user
As a data scientist, I want to ask questions of the hydrophone dataset being captured, so I can turn exploratory analyses into reusable metrics, models, and visual tools that others can query and validate.

#### Platform/product user
As an Orcasound platform builder, I want to ask questions of the hydrophone dataset being captured, so I can translate scientific analyses into durable APIs, dashboards, and interactive tools instead of one-off presentations.

#### Public-facing educator / communicator
As a science communicator, I want to ask questions of the hydrophone dataset being captured, so I can make complex acoustic patterns understandable to non-specialists without stripping away the underlying evidence.

#### If you want one umbrella version
As a scientist or conservation practitioner, I want to ask questions of the hydrophone dataset being captured, so I can turn raw underwater sound into interpretable evidence about vessel noise, masking, and orca habitat conditions.



The product stance here is:

1. build the data pipeline and API so the system can answer real questions
2. expose a set of exploratory tools grounded in scientific practice
3. observe which views and analyses actually produce insight
4. then harden those into productized workflows

That is still product development.
It is just evidence-first product development for a scientific domain.

**A useful way to describe the value**

You can say the PSD/parquet advancement improves Orcasound by shifting the platform from:

- static image generation and coarse event review

toward:

- interactive acoustic analysis over structured time-frequency data

That enables:

- richer interpretation
- faster iteration on scientific questions
- better linkage between vessel activity, noise exposure, and masking risk


### What the higher-fidelity PSD/WebGL path materially enables

**For users**

It can make the platform better by supporting:

- faster exploration of long time windows without waiting on regenerated PNGs
- clearer inspection of time-frequency structure at multiple scales
- more adaptable visualization for different audiences and questions
- overlays of non-acoustic context like AIS vessel tracks or vessel passages
- side-by-side comparison of raw noise, filtered bands, masking estimates, and anomalies
- more trustworthy analysis because the frontend is driven by numeric data, not only flattened images


**For researchers and analysts**

It can enable:

- band-specific inspection of vessel noise and biologically relevant frequency ranges
dynamic thresholding to identify masking-risk periods
- anomaly highlighting to find unusual events worth review
- visual correlation between vessel passages and noise signatures
- better support for exploratory analysis, not just static review

**For conservation use**

It can help move toward questions like:

- which vessels are contributing the most noise at a hydrophone
- when does vessel noise overlap with frequencies relevant to orca communication
what fraction of time is acoustically masked
- whether mitigation measures reduce measured masking/noise burden over time
where acoustic conditions are worsening or improving

That is much closer to conservation decision support than “orca present / not present.”

**“What if looking at orca calls up close is not that useful?”**

Also possible.

Then the platform may turn out to be more valuable for:

- vessel-noise attribution
- masking analysis
- baseline shifts
- anomaly detection
- temporal monitoring
