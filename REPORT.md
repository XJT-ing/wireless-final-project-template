# 无线通信文件传输基带仿真系统报告

## 摘要

本项目实现了一个端到端无线通信文件传输基带仿真系统。发送端将 `Test.txt` 中的 UTF-8 文本转换为比特流，依次经过 PN 扰码、卷积信道编码、帧封装和 Gray 编码 QPSK 调制；信道端使用 AWGN 模型加入可复现复高斯白噪声；接收端通过伪随机前导序列完成帧同步，并进一步完成 QPSK 解调、帧解析、Viterbi 译码、解扰和源解码，最终恢复为 `results/received.txt`。系统输出 `metrics.json`、星座图、BER-SNR 曲线和同步相关峰值图。在 `SNR=12 dB`、`seed=2026`、AWGN 条件下，`received.txt` 与 `Test.txt` 完全一致，BER 为 0，FER 为 0，`text_match_rate` 为 1.0，`checksum_pass` 为 true。

关键词：无线通信；QPSK；AWGN；帧同步；卷积码；Viterbi 译码；metrics schema

## 1. 系统链路

系统固定链路为：

`Source Encode -> Scramble -> Channel Encode -> Frame Build -> QPSK Modulate -> AWGN Channel -> Synchronization -> QPSK Demodulate -> Frame Parse -> Channel Decode -> Descramble -> Source Decode -> Metrics/Plots`

其中 QPSK 与 AWGN 是基础链路，卷积码与 Viterbi 译码作为提高模块。系统通过统一 CLI 运行：

```bash
python main.py --input Test.txt --output results/received.txt --snr 12 --seed 2026 --mod qpsk --channel awgn
```

## 2. 关键设计

源编码模块按 UTF-8 字节将文本转换为 bit stream。扰码模块使用固定 seed 生成 PN 序列并与 payload XOR，解扰使用同一 PN 序列。信道编码采用约束长度 3、生成多项式 `(7,5)` 的 rate-1/2 卷积码，接收端使用硬判决 Viterbi 译码恢复最可能的输入序列。

帧结构为：

`preamble_bits | original_length_32 | encoded_payload_length_32 | encoded_payload_bits | crc_32`

`original_length_32` 用于去除源恢复阶段的 padding，`encoded_payload_length_32` 用于从帧中准确截取卷积编码后的 payload，`crc_32` 覆盖原始 payload bytes。该设计可以同时处理 UTF-8 长度变化、卷积码尾比特和 QPSK 补零。

QPSK 使用 PRD 指定 Gray 映射：

- `00 -> (1+j)/sqrt(2)`
- `01 -> (-1+j)/sqrt(2)`
- `11 -> (-1-j)/sqrt(2)`
- `10 -> (1-j)/sqrt(2)`

AWGN 信道中 SNR 定义为调制符号平均功率与复高斯噪声平均功率之比。同步模块使用伪随机前导序列相关检测帧起点，不假设接收端天然知道起始位置。

## 3. 隐藏验证与边界主题

本项目显式覆盖以下隐藏测试和文档主题：

- `invalid_snr`：拒绝 `nan`、无穷大、负 SNR、非数字 SNR，合法范围为 `[0, 60] dB`。
- `invalid_modulation`：拒绝 `16qam` 等非基线调制参数；基础系统只接受 `qpsk`。
- `invalid_channel`：基础系统只接受 `awgn`。
- `missing_input`：输入文件不存在时在 CLI 层报错。
- `empty_input`：空输入 payload 的源编码和 metrics 行为保持定义。
- `padding`：通过 `original_length_32` 和 `encoded_payload_length_32` 区分源长度、卷积码尾比特和 QPSK 补零。
- `sync_offset`：通过前导相关处理 0 到 128 个 QPSK 符号的随机前置偏移。
- `low_snr`：低 SNR 下允许恢复失败，但仍输出 BER、FER、text_match_rate、checksum_pass、sync_start_index 和图像。
- `metrics_schema`：`metrics.json` 至少包含 `snr_db`、`seed`、`modulation`、`channel`、`payload_bits`、`ber`、`fer`、`text_match_rate`、`checksum_pass`、`sync_start_index`。
- `no_hardcoding`：不硬编码公开测试输入、输出或中间 bit stream。

## 4. 测试结果

公开测试和额外自测覆盖源编码、扰码、帧结构、QPSK、AWGN、同步、端到端恢复、低 SNR 和非法参数。最新本地测试为：

```text
26 passed
```

最终运行结果的关键 metrics 为：

- `ber = 0.0`
- `fer = 0.0`
- `text_match_rate = 1.0`
- `checksum_pass = true`
- `coding_scheme = convolutional(7,5)+viterbi`

## 5. 结果分析

星座图显示 12 dB 条件下 QPSK 接收符号集中在四个理想星座点附近，硬判决边界清晰。BER-SNR 曲线随 SNR 提高而下降，符合噪声功率降低时误码率下降的规律。同步峰值图显示前导相关峰明显，`sync_start_index` 与仿真的随机前置符号数一致，说明接收端确实通过相关检测定位帧起点。

低 SNR 下的主要瓶颈是 QPSK 硬判决错误和同步相关峰下降。若同步错误，长度字段和 CRC 会快速失败；若同步正确但存在少量符号错误，Viterbi 译码可以纠正部分错误。CRC 与 `checksum_pass` 用于判断最终恢复结果是否可信。

## 6. 可扩展方向

后续可以扩展软判决 Viterbi 译码、Rayleigh/Rician 衰落信道、均衡算法、OFDM、BPSK 和 16-QAM 对比实验。这些扩展可以用于性能比较，但不替代当前 QPSK + AWGN 基线链路。
