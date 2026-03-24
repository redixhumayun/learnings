## Deep Neural Networks

### Multi-Layer Perceptron (MLP)

Here each layer is just y[i] = F(W * y[i-1]), where F is a non-linear function (eg: ReLU: max(x, 0))

### Convolution Neural Network (CNN)

Used mostly for images which have spatial locality. 

### Recurrent Neural Network (RNN)

The most popular implementation of this is long short-term memory (LSTM)

## TPU Architecture

Built with PCIe connectors to reduce deployment risk so it can hook into SATA disk slots.

Because of this architecture is more similar to an FPU coprocessor than a CPU. Host CPU sends instructions to the TPU over PCIe and TPU doesn't fetch it's own instructions from memory whereas GPU does.

Because PCIe is slow TPU instructions are CISC-style to minimize number of instructions sent over the bus.

```
Host CPU ←─ PCIe (16 GB/s) ─→ TPU ←─ DDR3 (14 GB/s×2) ─→ Weight DRAM
```

PCIe is a general interconnect bus that can connect to SSD's, GPU's etc. TPU targets inference only since training and inference have different hardware requirements.

## Links

1. [Making Deep Learning Go Brrr From First Principles](https://horace.io/brrr_intro.html)