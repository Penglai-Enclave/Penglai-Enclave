![Penglai logo](docs/images/penglai_logo.jpg)

Penglai is a set of security solutions based on Trusted Execution Environment.

This repo contains an overview of the whole project.

It currently supports RISC-V platforms, including both high-performant MMU RISC-V64 arch and MCU (RISC-V32, no MMU).

## Systems

Penglai contains a set of systems satisfying different scenarios.

- **Penglai-TVM**: it is based on OpenSBI, supports fine-grained isolation (page-level isolation) between untrusted host and enclaves. The code is maintained in [Penglai-TVM](#).
- **Penglai-MCU**: it supports Global Platform, and PSA now. Not open-sourced. Refer [Penglai-MCU](#) for more info.
- **Penglai-sPMP** based on OpenSBI for Nuclei devices is maintained in [Nuclei SDK](https://github.com/Nuclei-Software/nuclei-linux-sdk/tree/dev_flash_penglai_spmp).
- **Penglai-sPMP**: it utilizes our sPMP proposal to provide basic enclave functionalities, based on old bbl. The prototype is avaialble [here](https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP).


## Formal verification

We have built a framework, Pangolin, based on Serval, to formally verify the Penglai's secure monitor.

We have finished:

- The specifications on cross-enclave communication interfaces, i.e., acquire\_server\_enclave, enclave\_call, and enclave\_return.
- Code proof of the above three interfaces using refinement.

## Features

We highlight several features on Penglai which are novel over other TEE systems.

### Cross-enclave communication

TODO

### Secure file system

Penglai adopts xv6fs and littlefs as its file system, serving enclaves.
It utilizes the encryption libraries in SDK to encrypt/decrypt the data and provide integrity protections.


### Encryption library in SDK

Penglai utilizes the mbedtls as its encryption library.

### Support existing frameworks

- PSA: description TODO
- GP: description TODO

## License Details

Mulan Permissive Software License，Version 1 (Mulan PSL v1)
