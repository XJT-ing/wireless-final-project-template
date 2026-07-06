# Mock Test Report and Design Revision Record

## Mock Test 1: Frame Structure

Mock payload bits were passed through `build_frame` and `parse_frame`. The first issue was that QPSK padding can add an extra zero when the total frame length is odd. Revision: DESIGN.md now states that the `length_32` field records the original payload bit count and the receiver trims payload bits by this value.

## Mock Test 2: Synchronization

A synthetic received vector was created with 25 random leading QPSK symbols, followed by the preamble and payload. The risk was assuming the receiver naturally knows frame start. Revision: the receiver now runs correlation over the whole received vector and records `sync_start_index` in metrics.

## Mock Test 3: End-to-End Recovery

The full mock chain used UTF-8 source bits, Scramble, convolutional Channel Encode, Frame Build, QPSK, AWGN, Synchronization, demodulation, Viterbi Channel Decode, descramble, and Source Decode. The defect found was that CRC must be computed on original payload bytes, not encoded bits, otherwise a decoder padding change can break checksum comparison. Revision: the CLI validates CRC after channel decoding and descrambling against the original payload bytes.

## Mock Test 4: Low-SNR Failure Behavior

At low SNR the design may produce bit errors or CRC failure. Revision: the CLI always writes `metrics.json` and plots even when text is not perfectly recovered, preventing failure cases from hiding diagnostic information.

## Mock Test 5: CLI Boundary and Hidden-Validation Topics

Mock CLI calls were added for invalid SNR and unsupported modulation values. The issue found was that accepting `nan`, infinity, negative SNR, or unsupported modulation would make generated metrics meaningless. Revision: `main.py` now validates `--snr` as a finite value in `[0, 60] dB`, restricts `--mod` to `qpsk`, restricts `--channel` to `awgn`, and checks that the input file exists before running the communication chain.

The test plan was also updated to mention `invalid_snr`, `invalid_modulation`, `missing_input`, `empty_input`, `padding`, `metrics_schema`, `sync_offset`, and `low_snr` as explicit hidden-validation topics.
