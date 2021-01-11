<img src="docs/images/Penglai-Logo-v2.png" width="200">

Penglai is a set of security solutions based on Trusted Execution Environment.

This repo contains an overview of the whole project.

It currently supports RISC-V platforms, including both high-performant MMU RISC-V64 arch and MCU (RISC-V32, no MMU).

## Systems

Penglai contains a set of systems satisfying different scenarios.

- **Penglai-TVM**: it is based on OpenSBI, supports fine-grained isolation (page-level isolation) between untrusted host and enclaves. The code is maintained in [Penglai-TVM](#).
- **Penglai-MCU**: it supports Global Platform, and PSA now. Not open-sourced. Refer [Penglai-MCU](#) for more info.
- **Penglai-sPMP**: it utilizes our sPMP (S-mode PMP) proposal to provide basic enclave functionalities. A version based on OpenSBI for Nuclei devices is maintained in [Nuclei SDK](https://github.com/Nuclei-Software/nuclei-linux-sdk/tree/dev_flash_penglai_spmp). Another prototype based on old BBL (with QEMU) is avaialble [here](https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP).


## Features

We highlight several features on Penglai which are novel over other TEE systems.

### Cross-enclave communication

Penglai supports the synchronous and asynchronous cross-enclave communication using the zero-copy memory transferring mechanism.

##### Synchronous IPC interface:

```c
struct call_enclave_arg_t
{
  unsigned long req_arg;
  unsigned long resp_val;
  unsigned long req_vaddr;
  unsigned long req_size;
  unsigned long resp_vaddr;
  unsigned long resp_size;
};

unsigned long acquire_enclave(char* name);;
unsigned long call_enclave(unsigned long handle, struct call_enclave_arg_t* arg);
void SERVER_RETURN(struct call_enclave_arg_t *arg);
```

As for synchronous cross-enclave communication, Penglai sdk provides three IPC-related interfaces: `acquire_enclave`, `call_enclave` and `SERVER_RETURN`.

+ `Acquire_enclave` receives the callee enclave name and returns the corresponding enclave handler.

+ `Call_enclave` has two parameters, one is the enclave handler, and another is the argument's structure: `struct call_enclave_arg_t`. `struct call_enclave_arg_t` contains six variables: `req_arg` and `resp_val` are the calling and return value passing by the register. `req_size` and `req_vaddr` indicate a memory range that will map to the callee enclave using the zero-copy mechanism. `resp_size` and `resp_vaddr` are similar to the `req_size` and `req_vaddr`,  which are ignored in the calling procedure, but will be filled by monitor in the return procedure.

  Call_enclave will be hanged until the calling procedure returns.

+ `SERVER_RETURN` also receives the `struct call_enclave_arg_t`, which indicates the `resp_size` and `resp_vaddr`.

Asynchronous IPC interface:

```c
unsigned long asyn_enclave_call(char* name, struct call_enclave_arg_t *arg);
```

`asyn_enclave_call` receives two parameters, one is the callee enclave name and another is the pointer of the `struct call_enclave_arg_t`, but only the req_vaddr and `req_size` are effective. When an enclave invokes the `asyn_enclave_call`, Penglai monitor will unmap these memory pages, and re-map them to the callee enclave before it running. In addition, asyn_enclave_call will not suspend the caller enclave procedure.

### Secure file system

Penglai adopts xv6fs and littlefs as its file system, serving enclaves.
It utilizes the encryption libraries in SDK to encrypt/decrypt the data and provide integrity protections.

Penglai add an interface layer on top of file system to support unified interface abstraction to client enclave. In the implementation of this layer, these file system related interfaces are the functions supported to client. Clients call the functions in the way of IPC between enclave.

On the client side, we hide the details of IPC in the musl libc we offered in penglai SDK. So if a client want to interact with file system, they can just use the standard libc file system interfaces.

At this moment, the following file system related interfaces are supported.

```c
FILE* fopen(const char* pathname, const char* mode);
FILE* fclose(FILE* stream);
size_t fread(void* ptr, size_t size, size_t nmemb, FILE* stream);
size_t fwrite(const void* ptr, size_t size, size_t nmemb, FILE* stream);
char* fgets(char* s, int size, FILE* stream);
int fputs(const char* s, FILE* stream);
int fseek(FILE* stream, long offset, int whence);
int mkdir(const char* pathname, mode_t mode);
int stat(const char* path, struct stat *buf);
int truncate(const char* path, off_t length);
```

Here is a easy example to show how to use file system.
```c
int EAPP_ENTRY main(){
    unsigned long* args;
    EAPP_RESERVE_REG;
    FILE *f;
    int status;
    char buf[52] = {0};
    eapp_print("before fopen\n");
    f = fopen("/create.txt","w");
    char str[] = "just for create and write\ntest fgets";
    fputs(str,f);
    eapp_print("after fwrite\n");
    struct stat st;
    fclose(f);
    status = stat("/create.txt", &st);
    if(!status){
        eapp_print("stat succeed: file length: %d\n",st.st_size);
    }
    f = fopen("/create.txt","r");
    memset(buf,0,52);
    eapp_print("fopen /create.txt for read\n");
    char* res = fgets(buf,52,f);
    if(res == NULL){
        eapp_print("fgets failed\n");
    }
    eapp_print("read writed content: %s\n",buf);
    fseek(f,0,SEEK_SET);
    memset(buf,0,53);
    fgets(buf,52,f);
    eapp_print("read after seek: %s\n",buf);
    fclose(f);
    f = fopen("/sub/empty.txt","w");
    if(!f){
        eapp_print("open file failed, directory does not exist\n");
        if(mkdir("/sub", 0) != 0){
            eapp_print("mkdir failed\n");
            EAPP_RETURN(0);
        }
    }
    f = fopen("/sub/empty.txt","w");
    if(!f){
        eapp_print("open after mkdir failed\n");
        EAPP_RETURN(0);
    }
    fputs(str, f);
    fclose(f);
    eapp_print("write to empty file\n");
    f = fopen("/sub/empty.txt","r");
    fgets(buf,52,f);
    eapp_print("read from empty file, %s\n",buf);
    fclose(f);
    int fd;
    if ((fd = open("/sub/empty2.txt", O_RDWR| O_CREAT)) < 0) {
        eapp_print("open error\n");
     }
    struct stat stat_buf;
    eapp_print("stat begin\n");
    stat("/sub/empty.txt", &stat_buf);
    printf("/sub/empty.txt file size = %d/n", stat_buf.st_size);
    EAPP_RETURN(0);
}
```

### Encryption library in SDK

Penglai supports national secret algorithm like **SM4**.

Besides, Penglai utilizes the mbedtls to provide other crypto algorithm support.

Mbed TLS is a C library that implements cryptographic primitives, X.509 certificate manipulation and the SSL/TLS and DTLS protocols. Its small code footprint makes it suitable for embedded systems. And Mbed TLS includes a reference implementation of the PSA Cryptography API.

You can find the basic doc tutorial in mbedtls/docs. And if you want to get more specific API documents, you can use doxygen engine to generate the API documents. The doxygen file is located at mbedtls/doxygen.

The implementation of mbedtls is very flexible and very easy to port to different platforms. You can modify the configurations in mbedtls/include/mbedtls/config.h to adapt to the library to your platform.




### Support existing frameworks

- PSA: description TODO
- GP: description TODO

## License Details

Mulan Permissive Software License，Version 1 (Mulan PSL v1)
