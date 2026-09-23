# ABR Auditory Stimulus Project

A low-cost Auditory Brainstem Response (ABR) hearing screener, built as the final design project for BioE 101 (Bioinstrumentation) at UC Berkeley.

The goal was to detect the ABR, a roughly 0.1 to 1 microvolt neural signal evoked by sound, using an analog front end, an Arduino based data acquisition stage, and coherent averaging in software. ABR is the clinical gold standard for screening newborns for congenital hearing loss, and commercial units cost between $8,000 and $20,000. This project reproduces the core measurement chain with parts costing a small fraction of that, and shows a recorded signal that responds to an auditory stimulus.

> Course: BioE 101, UC Berkeley (Prof. Conolly)
> Type: Capstone / final design project
> Status: Prototype, signal validated against the ABR gold standard waveform

---

## Table of contents

1. [Motivation](#motivation)
2. [What an ABR device does](#what-an-abr-device-does)
3. [Design goals and specifications](#design-goals-and-specifications)
4. [System architecture](#system-architecture)
5. [Component selection](#component-selection)
6. [Circuit and gain budget](#circuit-and-gain-budget)
7. [Noise and interference model](#noise-and-interference-model)
8. [Data acquisition, anti-aliasing, and the ADC](#data-acquisition-anti-aliasing-and-the-adc)
9. [Results](#results)
10. [Tradeoffs and lessons learned](#tradeoffs-and-lessons-learned)
11. [Repository contents](#repository-contents)
12. [Team](#team)
13. [References](#references)

---

## Motivation

Between 1 and 3 out of every 1,000 babies are born with permanent hearing loss. Undetected, this stunts social and language development at a critical early age, because a child who cannot hear a parent's voice misses the window in which speech and language normally take root. Early screening lets clinicians intervene with hearing aids or cochlear implants while that window is still open.

## What an ABR device does

An ABR test measures the electrical activity of the auditory pathway in response to a sound stimulus. It is well suited to newborns because an infant cannot give a reactionary gesture such as raising a hand when a tone is heard, so a passive, evoked measurement is needed.

A healthy response contains a series of characteristic peaks (waves I through VII). Wave V is the clinically important landmark. When the stimulus level is lowered past roughly 70 dB, wave V is the last peak to disappear in a normal-hearing patient, and its absence is what confirms congenital hearing loss. The screener compares the recorded response against this expected waveform.

![ABR explanation and normal versus hearing-loss response](docs/slides/slide-03.jpg)

## Design goals and specifications

The physiologic requirements of the ABR signal drive every engineering choice: the signal is tiny, buried in 60 Hz mains interference, and only reliably recovered by averaging thousands of trials.

| Physiologic need | Engineering specification |
|---|---|
| Detect wave V | Bandwidth 100 Hz to 3 kHz |
| Biosignal of 0.1 to 1 microvolt | Total gain = 10^5 (100 dB) target |
| Reject 60 Hz EMI | CMRR > 100 dB |
| Wave V latency 5 to 7 ms | Sampling rate f_s = 20 kHz (much greater than 2 x BW) |
| Quantization below the noise floor | ADC 12 bits (LSB = 2.4 mV, well below the 280 mV output sigma) |
| Usable output SNR | Target SNR > 20 dB after averaging |

Real ABR instruments average on the order of 2,000 trials to pull the response out of the noise, since averaging suppresses the noise contributed by every filtering stage. SNR improves with the square root of the number of averages.

![Engineering design goals and signal chain block diagram](docs/slides/slide-04.jpg)

## System architecture

The end to end chain is:

```
Electrodes  ->  Bandpass amplifier  ->  Anti-aliasing filter  ->  ADC  ->  Computer
             (AD620 IA + LM324 HPF/LPF        (Sallen-Key)      (Arduino)   (FFT +
              + 60 Hz Twin-T notch)                                          averaging)
```

Electrodes sit at the forehead and behind the ear. The analog front end amplifies and band-limits the signal, an anti-aliasing filter protects the sampler, the Arduino digitizes the waveform, and a Python pipeline performs the FFT, spectrogram, and coherent averaging.

## Component selection

The instrumentation amplifier sets the noise floor for the whole system, so the first stage uses an AD620 rather than a general purpose op-amp. The tradeoff is cost against noise and common-mode rejection.

| Spec | AD620 (instrumentation amp) | LM324 (quad op-amp) |
|---|---|---|
| Bandwidth | 120 kHz | 1 MHz |
| Input voltage noise (@ 1 kHz) | 9 nV/sqrt(Hz) | 35 nV/sqrt(Hz) |
| Input current noise | 0.1 pA/sqrt(Hz) | 0.01 pA/sqrt(Hz) |
| CMRR | 110 to 130 dB (G > 1000) | 65 to 80 dB |
| Unit cost | $13 to $18 | $0.13 to $0.60 |

The AD620 is used for the first stage, where its high CMRR and low voltage noise matter most. The much cheaper LM324 is used for the later gain and filtering stages, where the signal has already been amplified.

## Circuit and gain budget

The signal chain is: AD620 instrumentation amplifier at gain 10, an LM324 high-pass filter stage at gain 200 with a corner near 100 Hz, an LM324 low-pass filter stage at gain 200 with a corner near 3 kHz, and a passive 60 Hz Twin-T notch filter at unity gain.

```
Total gain = 10 x 200 x 200 x 1 = 400,000
Passband   = 100 Hz to 2,950 Hz
```

![Hand-drawn circuit diagram with stage gains and cutoffs](docs/slides/slide-06.jpg)

## Noise and interference model

The model estimates signal and noise amplitudes and SNR directly at the sensor output, and it explains why component placement matters so much.

- AD620 source impedance: 90 kOhm
- LM324 source impedance: 4 MOhm
- Thermal noise of the largest resistor (200 kOhm): 3.1 microvolts
- Noise figure, AD620 stage: about 1.1, giving an SNR loss of 0.45 dB
- Noise figure, LM324 stage: about 48, giving an SNR loss of 16.81 dB
- 60 Hz mains coupling is the dominant interference, handled by the Twin-T notch filter

The large noise figure of the LM324 is the reason it is kept out of the first stage. Placed early, its noise would dominate. Placed late, it acts on an already amplified signal.

## Data acquisition, anti-aliasing, and the ADC

Getting a clean digital record turned out to be the hardest part of the build.

- The amplified noise saturated the Arduino input, which made the raw ABR impossible to see before averaging.
- Memory limits on the microcontroller forced a practical sampling rate near 6 kHz and made large-N averaging difficult to hold in memory.
- Quantization step in the built prototype: full scale over 2^B = 5 V / 2^10 = 4.88 mV.
- A Sallen-Key anti-aliasing filter with a 3 kHz corner sits between the analog output (V_in) and the Arduino input (V_out).

## Results

The recorded oscilloscope waveform shows peaks in roughly the expected positions for waves I through VII when compared against the ideal ABR waveform. The signal clearly responds to the auditory stimulus, which validates the core measurement concept, but it carries a large amount of artifact.

```
SNR estimation = 20 * log10(V_rms / noise_std)
               = 20 * log10(7.071 / 0.3)
               = 27.5 dB
```

The high artifact content is consistent with the noise budget and, as the vibration testing showed, with mechanical coupling into the electrodes.

![Recorded oscilloscope waveform next to the ideal ABR waveform](docs/slides/slide-10.jpg)

A separate 20 Hz to 20 kHz electrode vibration test showed strong response as the tone approached 20 kHz, pointing to skull and bone vibration as a likely source of the recorded artifacts.

## Tradeoffs and lessons learned

- Dry versus wet electrodes: dry electrodes still needed gel to make contact, and wet electrodes worked just as well, so the convenience advantage of dry electrodes did not materialize here.
- LM324 noise compounds at every gain stage, which reinforced the decision to reserve the low-noise AD620 for the first stage.
- The ABR amplitude is extremely small relative to skull vibration, so mechanical isolation matters as much as electrical filtering.
- The same fundamentals apply as in an EKG measurement. The ABR is just a much smaller signal, which makes every noise and interference decision more consequential.

## Repository contents

```
abr-hearing-screener/
├── README.md
├── LICENSE
├── .gitignore
└── docs/
    ├── ABR_Auditory_Stimulus_Project.pptx   Full presentation (source)
    ├── ABR_Auditory_Stimulus_Project.pdf    Full presentation (viewable without PowerPoint)
    └── slides/                              Individual slide images used in this README
        ├── slide-01.jpg
        ├── ...
        └── slide-15.jpg
```

The complete slide deck is in `docs/`, in both PowerPoint and PDF form. The individual slide images embedded above live in `docs/slides/`.

## Team

- Kingshuk Daschowdhury
- Eins Besmanos
- Achyut Chebiyam
- Jack Langhoff
- Tanya Hemdev

Course project for BioE 101 (Bioinstrumentation), UC Berkeley.

## References

1. Analog Devices, "Low Cost Low Power Instrumentation Amplifier AD620," Rev. H, Jul. 2011.
2. Texas Instruments, "LMx24-N, LM2902-N Low-Power, Quad-Operational Amplifiers," SNOSC16D, Jan. 2015.
3. "Trial-By-Trial Auditory Brainstem Response Detection." bioRxiv, February 3, 2026. https://doi.org/10.64898/2026.01.31.703019
4. "Cost-effectiveness of portable-automated ABR for universal neonatal hearing screening in India." Frontiers in Public Health, 12. https://doi.org/10.3389/fpubh.2024.1364226
5. Ma, J., Choi, S. J., Kim, S., and Hong, M. (2024). "Performance comparison of convolutional neural network-based hearing loss classification model using auditory brainstem response data." Diagnostics, 14(12), Article 1232. https://doi.org/10.3390/diagnostics14121232
