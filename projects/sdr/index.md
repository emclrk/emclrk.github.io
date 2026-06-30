---
layout: default
title: SDR Garage Door Signal Analysis
---

[← Projects](/projects/)

# SDR Garage Door Signal Analysis

<div class="tags" style="margin:0.5rem 0 2rem">
  <span class="tag">RF</span>
  <span class="tag">signals</span>
  <span class="tag">SDR</span>
  <span class="tag">reverse engineering</span>
  <span class="tag">OOK</span>
</div>

**Tools:** Nooelec NESDR SMArt RTL-SDR · gqrx · Universal Radio Hacker · Python (pylcs, ad hoc analysis scripts)

---

## Introduction / Motivation

I was at home one day when my garage door opened seemingly of its own accord — nobody in our house triggered it. This happened 3 or 4 times that day, and never again. I found this deeply disconcerting. There was no obvious sign of malfunction, and knowing that garage door remotes are notoriously insecure, I wondered how likely it was that someone was driving through my neighborhood with a transmitter to see which garages they could open. Then I wondered if that would even be possible for my system specifically — and decided to investigate and attempt to evaluate how secure my system is. What does the signal look like? Does it have security mechanisms such as a rolling code? How difficult would it be to replicate or spoof the signal?

## Setup

### Hardware

Nooelec NESDR SMArt RTL-SDR. Tuning range: 100 kHz – 1.75 GHz (advertised).
Telescopic antenna — not optimizing for antenna performance here. Good enough is good enough.
The remote brand is Craftsman with FCC ID HBW1255, manufactured in 2002.

### Software

I used gqrx to capture the signals and Universal Radio Hacker (URH) to process them. Demodulation experiments on the bit strings were done with ad hoc Python scripts.

## Finding the Signal

Various unverified internet sources identified typical garage door operating frequencies between 300–400 MHz, varying by year of manufacture. Specific frequencies mentioned included 310 MHz, 315 MHz, 390 MHz, and 433 MHz. (Notably, 433 MHz is in the 70 cm UHF band.) I found the FCC ID and looked it up in the FCC database. I confirmed the nominal operating frequency is 390 MHz and noted that the authorization was granted in 1997 — old enough that the signal will likely be transparent, though 1997 is also right when rolling code garage door openers were first being introduced.

I plugged the RTL-SDR into my laptop and opened gqrx. I checked a couple of bands while repeatedly pressing the remote button and saw activity around 390 MHz — a bursty signal.

## Modulation and Symbol Rate

Internet sources seem to agree that garage door remotes usually use Amplitude Shift Keying (ASK) or On-Off Keying (OOK). Cost is the primary driving factor behind the modulation choice — OOK is very simple, meaning cheap receiver hardware. Since the system is low data rate there is no need to worry about spectral efficiency, and the noisiness that tends to affect amplitude modulation schemes is not much of a problem at short range. Short-range, line-of-sight transmission has a reasonably high SNR, so amplitude noise is insignificant.

I took several recordings of I/Q data — single button presses, multiple button presses in sequence (to determine if there's a rolling code), and a continuous button press. Sample rate: 2.4 M samples per second.

First observation: in the waterfall plot, the signal occupies a wider band than expected — enough that I initially wondered if it might be FSK. Some of this spectral messiness is due to the sharp on/off of the transmitted signal (a rectangular pulse in the time domain maps to a sinc pattern in the frequency domain). The rest appears to be hardware artifacts, since there are multiple images of the same signal on the spectrogram.

![I/Q Images — one button press](figs/spectrogram_images.png)

I'm seeing 8 distinct bursts of activity per button press.

Some questions: what are the bursts? How many symbols per burst?

Here's a zoomed-in look at the signal view demodulation:

![Zoomed demodulation](figs/zoomed_in_demod.png)

On-Off Keying encodes data by either transmitting a carrier wave or not transmitting. I'm confident this is OOK.

Interestingly, the pulses and gaps between pulses all appear to be approximate multiples of 500 μs (0.5 ms). I'm going to hypothesize that a symbol is 500 μs long. With a sample rate of 2.4 M samples/s and 500 μs per symbol, there should be 1,200 samples per symbol. I set the samples/symbol value in URH accordingly.

With these parameters, URH showed:

![Initial demodulation](figs/demod_init.png)

Some good signs: one line of bits corresponds to one of the 8 bursts, and there are long strings of repeated bits between bursts 1 and 3 and between bursts 2 and 4 — a good sign that the structure is becoming clear.

Zooming in further:

![One bit](figs/one_bit.png)

One bit corresponds to one 500 μs chunk — a 1 for each period containing a pulse and a 0 for each period not containing a pulse.

Looks like each button press contains 8 bursts, and the 8 bursts are really two patterns that alternate: bursts 1, 3, 5, and 7 are identical (pattern A, 81 bits), and bursts 2, 4, 6, and 8 are also identical (pattern B, 83 bits). Patterns A and B change slightly between button presses — a good thing to investigate next.

One notable pattern that is NOT present: there are no strings of 0s or 1s longer than 3 bits. This is interesting because of a patent I found by Chamberlain (a common maker of garage door openers), submitted in 1997:

> A rolling code transmitter is useful in a security system for providing secure encrypted RF transmission comprising an interleaved *trinary* bit fixed code and rolling code.

The patent goes on to describe fixed-width pulses representing 0, 1, and 2 — this could be what I'm seeing. (The brand is Craftsman, but Chamberlain has manufactured Craftsman openers for years under a licensing agreement.)

From the patent:
> wherein a first of the different configurations is represented by a first amplitude radio frequency signal having a length of about 0.5 milliseconds followed by a second amplitude radio frequency signal having a length of about 1.5 milliseconds,
>
> wherein a second of the different configurations is represented by a first amplitude radio frequency signal having a length of about 1.0 milliseconds followed by a second amplitude radio frequency signal having a length of about 1.0 milliseconds,
>
> wherein a third of the different configurations is represented by a first amplitude radio frequency signal having a length of about 1.5 milliseconds followed by a second amplitude radio frequency signal having a length of about 0.5 milliseconds.

This confirms the 500 μs symbol length hypothesis.

## Initial Bit Analysis

Looking closer at the actual bits from two button presses, in hex:

```
(A) 8bb8899b9b8bbbb899b98
(B) ee2eee22e222226e62ee2
(A) 8bb8899b9b8bbbb899b98    ...repeated 4× per press
```

```
(A) 9898b8bb9b8b9999898b8
(B) ee266ee6eeee62e66622e
(A) 9898b8bb9b8b9999898b8    ...repeated 4×
```

The sequences look similar but not identical. My understanding is that rolling codes in these systems are provided by pseudo-random number generators. But the hex codes look almost too similar. On the other hand, I remember the earlier language about ternary symbols — perhaps there are fewer possible symbols, and my intuition was assuming we could see any combination of bits.

The same two button presses in octal (3-bit) representation:

```
(A) 427342114671561356734231563
(B) 7342735610561042104671427341
```

```
(A) 461142705671561346314611427
(B) 7342315671567356305631461053
```

Not clearer. Another approach: running correlations between strings from different button presses, or looking at the long-press recording.

Looking at the long press (holding the button continuously), the bits were these two lines repeated 19 times each:

```
(A) 98b98bb9999b99999bbb8
(B) ee266ee6e2222266226ee
```

No variation at all in the long press. Strange — maybe the transmitter just kept restarting the same transmission, or the rolling code only increments on a button release.

I computed the entropy of the bits using different symbol lengths. The entropy for 4-bit symbols is notably lower than for 2, 3, or 5 bits — consistent with a constrained 4-bit alphabet.

## Protocol Structure

I recorded a new dataset with 10 presses to get more data. Each press followed the same A/B/A/B pattern, and no burst strings repeated between presses — suggesting a rolling code.

Visual inspection of the hex strings showed a constrained alphabet:
- A bursts: only symbols `0x8`, `0x9`, `0xb` (and occasionally `0x1`) appear
- B bursts: only `0x2`, `0x6`, and `0xe` appear

Longest common substring analysis (using pylcs):
```
Common substring of all A bursts: 10001011100110  (length 14)
Common substring of all B bursts: 1001100010      (length 10)
```

These seem long enough to be synchronization words — but they don't appear at a consistent position in each burst, so this approach was a dead end.

However, grouping the A and B bursts and removing duplicates revealed:
- All A bursts begin with `100`
- All B bursts begin with `1110`

The structure of an A burst:

![A burst protocol](figs/Burst_a_bit_protocol.png)

The first three bits are `100`. The fourth bit varies. Then a repeating pattern: two bits equal to `10` followed by two varying bits. The final bit (bit 81) is always `1`. In shorthand: `100X 10XX 10XX .... 10XX 1`

![B burst protocol](figs/burst_b_protocol.png)

Same alternating pattern in the B bursts: `11 10XX 10XX 10XX .... 10XX 1`

Extracting the varying bits and counting 2-bit symbols:

```
A counts: {'00': 60, '01': 53, '10': 0, '11': 77}
B counts: {'00': 66, '01': 71, '10': 0, '11': 63}
```

The complete absence of `10` in the varying bits confirms a ternary alphabet: symbols `1000` (0x8), `1001` (0x9), and `1011` (0xb). The `10` pattern serves as a per-symbol header, likely helping the receiver stay synchronized. This wasn't immediately visible because the symbols don't start at bit 1 — they start at bit 5 on A bursts and bit 3 on B bursts.

The A/B repetition pattern (each burst transmitted 4 times) appears to be a built-in noise protection: if a bit flip corrupts one burst, the receiver discards it and uses one of the duplicates.

I speculated that bit 4 in the A bursts might be a checksum, but summing the varying bits mod 2 didn't support that hypothesis. Beyond this, it's unclear which changing bits are payload vs. CRC/checksum.

## Conclusion

An interesting consequence of this process was discovering a packet structure different from what I expected. I anticipated clear demarcations between header, payload, and checksum. Instead the payload is interleaved with known symbols — a reasonable design, but a point of learning that will give me a broader mental model of protocol structures.

The turning point in making sense of the structure came from removing duplicate burst strings and noticing that A and B bursts always started with the same bits. The structure then became clear: a short prefix, followed by four-bit symbols starting with `10`. This is a ternary alphabet consisting of `1000` (0x8), `1001` (0x9), and `1011` (0xb) — exactly as suggested by the Chamberlain patent.

More information — the presence or absence of a checksum or CRC — could likely be inferred by looking more closely at the patent documentation or by brute-forcing a variety of CRC lengths.

The presence of a rolling code makes me more confident in the overall security of the opener. Earlier implementations lacked this, making it easy to drive around the neighborhood and trigger any garage. The Chamberlain patent also describes additional receiver-side protections: if the rolling code counter is much further ahead than the receiver expects, two consecutive correct transmissions are required to respond.

Newer systems have even more protections. This older system is probably still vulnerable to attacks like Samy Kamkar's RollJam (DEF CON 23) — which jams the signal to prevent it reaching the receiver while simultaneously recording the transmission, capturing a valid code to replay later. In the end, I conclude that while not unbreakable, the system is likely secure enough for most purposes. Though it may be wise to store important valuables in a more secure location.

---

*AI disclosure: generative AI was used to proofread this document for spelling errors and correct grammar, and to assist in formatting the final document. All research and analysis was done by me.*
