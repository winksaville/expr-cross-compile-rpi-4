# Experiment with cross compiling for Raspberry-Pi 4

Roughly followed these [instructions](https://medium.com/swlh/compiling-rust-for-raspberry-pi-arm-922b55dbb050)
but instead of using `target.arm7-linux-gnueabihf` I used `target.aarch64-unknown-linux-gnu`.
I determined this by installing the native gcc compiler and use readlink to see what it pointed to:
```
$ readlink -f `which gcc`
/usr/bin/aarch64-linux-gnu-gcc-11
```
Also you can use the native `gcc -v` looking for Target:
```
wink@rpi48:~/prgs/c/hw
$ gcc -v 2>&1 | grep Target:
Target: aarch64-linux-gnu
```
Another technique was to create hw.c and compile it then look at the `file` output:
```
wink@rpi48:~/prgs/c/hw
$ cat hw.c
#include <stdio.h>

int main() {
  printf("Hello World\n");
  return 0;
}
wink@rpi48:~/prgs/c/hw
$ gcc hw.c -o hw
wink@rpi48:~/prgs/c/hw
$ file hw
hw: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, BuildID[sha1]=8b57a8e0b5dd0a6495f705886636af06ef1fd964, for GNU/Linux 3.7.0, not stripped
```

So I installed aarch64 tools:
```
sudo pacman -Syu aarch64-unknown-linux-gcc aarch64-unknown-linux-binutils
```

And here is `.cargo/config`:
```
wink@3900x:~/prgs/rust/myrepos/expr-cross-compile-rpi-4 (main)
$ cat .cargo/config.toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
```

Here is the `deploy` executable which builds and runs hw:
```
wink@3900x:~/prgs/rust/myrepos/expr-cross-compile-rpi-4 (main)
$ cat deploy
#!/usr/bin/env bash
# From:
#  https://medium.com/swlh/compiling-rust-for-raspberry-pi-arm-922b55dbb050

# Make bash stricter: https://kvz.io/bash-best-practices.html
set -o errexit
set -o nounset
set -o pipefail
set -o xtrace

# Target parameters
readonly TARGET_HOST=wink@rpi48
readonly TARGET_PATH=/home/wink/bin/hw
readonly TARGET_ARCH=aarch64-unknown-linux-gnu
readonly SOURCE_PATH=./target/${TARGET_ARCH}/release/hw

# Build
cargo build --release --target=${TARGET_ARCH}

# Deploy
scp ${SOURCE_PATH} ${TARGET_HOST}:${TARGET_PATH}

# Run
ssh -t ${TARGET_HOST} ${TARGET_PATH}
```

Run `deploy` to build and run:
```
wink@3900x:~/prgs/rust/myrepos/expr-cross-compile-rpi-4 (main)
$ ./deploy
+ readonly TARGET_HOST=wink@rpi48
+ TARGET_HOST=wink@rpi48
+ readonly TARGET_PATH=/home/wink/bin/hw
+ TARGET_PATH=/home/wink/bin/hw
+ readonly TARGET_ARCH=aarch64-unknown-linux-gnu
+ TARGET_ARCH=aarch64-unknown-linux-gnu
+ readonly SOURCE_PATH=./target/aarch64-unknown-linux-gnu/release/hw
+ SOURCE_PATH=./target/aarch64-unknown-linux-gnu/release/hw
+ cargo build --release --target=aarch64-unknown-linux-gnu
    Finished release [optimized] target(s) in 0.00s
+ scp ./target/aarch64-unknown-linux-gnu/release/hw wink@rpi48:/home/wink/bin/hw
hw                                                                                    100% 3700KB  59.3MB/s   00:00    
+ ssh -t wink@rpi48 /home/wink/bin/hw
Hello, world!
Connection to 192.168.1.112 closed.
```

And to double check log into the rpi and validate:
```
wink@rpi48:~
$ which hw
/home/wink/bin/hw
wink@rpi48:~
$ hw
Hello, world!
wink@rpi48:~
$ sha256sum bin/hw
3bbf2bf52c3a4049fb3e2d6fcc91c8c09185a5cfe209d0f9e742c0cb90329760  bin/hw
```

And then on the source machine validate the `sha256sum` is the same:
```
wink@3900x:~/prgs/rust/myrepos/expr-cross-compile-rpi-4 (main)
$ sha256sum target/aarch64-unknown-linux-gnu/release/hw
3bbf2bf52c3a4049fb3e2d6fcc91c8c09185a5cfe209d0f9e742c0cb90329760  target/aarch64-unknown-linux-gnu/release/hw
```

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.
