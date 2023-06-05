| Supported Targets | ESP32-S3 |
| ----------------- | -------- |

# GLOW Image Recognition Demo

This is a demo project for testing [GLOW Ahead Of Time (AOT) compiled executable bundles](https://github.com/pytorch/glow/blob/master/docs/AOT.md). This project is based on ESP IDF [file server example](https://github.com/espressif/esp-idf/tree/master/examples/protocols/http_server/file_serving). Follow the link above for general description.

This project integrates GLOW AOT bundle built from [LeNet and MNIST handwritten digit recognition model](https://medium.com/mlearning-ai/lenet-and-mnist-handwritten-digit-classification-354f5646c590). Every PNG file uploaded to the server is passed to NN and the result is printed to terminal. [LeNet and MNIST bundle example](https://github.com/pytorch/glow/tree/master/examples/bundles/lenet_mnist) from GLOW project was used as a reference.
For PNG images processing [PNGdec library](https://github.com/bitbank2/PNGdec) was used.

## How to use the example

To test how NN works you can flash this app, connect to server and upload MNIST handwritten digit images from `tests/images/mnist`. Line like `Result: 2` shows recognized digit.

## Example Output

```
I (16642) file_server: Receiving file : /2_1065.png...
I (16642) file_server: Remaining size : 271
I (16642) file_server: File reception complete
I (16652) file_server: Process file '/data/2_1065.png'...
Load image: /data/2_1065.png
image specs: (28 x 28), 8 bpp, pixel type: 0
Loaded image: /data/2_1065.png
Loaded images size in bytes is: 3136
Copying image data into mutable weight vars: 3136 bytes
Result: 2
Confidence: 0.997419
I (17182) file_server: File processed successfully
I (17242) file_server: Found file : 0_1009.png (310 bytes)
I (17252) file_server: Found file : 1_1008.png (159 bytes)
I (17262) file_server: Found file : 2_1065.png (271 bytes)
I (26572) file_server: Receiving file : /9_1088.png...
I (26572) file_server: Remaining size : 269
I (26572) file_server: File reception complete
I (26582) file_server: Process file '/data/9_1088.png'...
Load image: /data/9_1088.png
image specs: (28 x 28), 8 bpp, pixel type: 0
Loaded image: /data/9_1088.png
Loaded images size in bytes is: 3136
Copying image data into mutable weight vars: 3136 bytes
Result: 9
Confidence: 0.946085
I (27112) file_server: File processed successfully
I (27162) file_server: Found file : 0_1009.png (310 bytes)
I (27182) file_server: Found file : 1_1008.png (159 bytes)
I (27202) file_server: Found file : 2_1065.png (271 bytes)
I (27222) file_server: Found file : 9_1088.png (269 bytes)
```

## TODO
- Avoid integrating constant weights from `lenet_mnist.weights.txt` [into app image](https://github.com/gerekon/glow_image_file_server/blob/main/main/img_nn.cpp#L282). It adds ~1.7 MB to flash image RO data (DROM). We can keep weights binary file `lenet_mnist.weights.bin` in SPIFFS and load it at startup. ESP32-S3 have 8MB PSRAM, so we can allocate buffer for it via `malloc`.
- Dynamically allocate large buffers for [mutable weights](https://github.com/gerekon/glow_image_file_server/blob/main/main/img_nn.cpp#L287) and [activations](https://github.com/gerekon/glow_image_file_server/blob/main/main/img_nn.cpp#L291) via `malloc`.
- Cleanup `PNGdec`. Make it as component and connect sources as GIT submodule.
- Add special button on HTML page to analize selected file
- Make LLVM Xtensa backend using TIE instructions in the GLOW bundle
- Port to ESP32-P4

## How to build everything from scratch

### Build LLVM
To build bundles for Espressif chips you need to build GLOW with our LLVM port. GLOW needs full LLVM framework built with enabled RTTI. LLVM also needs some modification to be compatible with GLOW project because LLVM 15.0.0 version broght some breaking changes. For LLVM you need to use [this branch](https://github.com/gerekon/llvm-project/tree/glow_port).
```bash
git clone -b glow_port --single-branch https://github.com/gerekon/llvm-project.git
export LLVM_PROJECT_PATH=$PWD/llvm-project
export LLVM_BUILD_DIR=$PWD/build_llvm
mkdir $LLVM_BUILD_DIR
cd $LLVM_BUILD_DIR
cmake -G Ninja $LLVM_PROJECT_PATH/llvm \
  -DCMAKE_INSTALL_PREFIX=$PWD/dist \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_RTTI=ON  \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="Xtensa"
cmake --build . --target install
```

### Build GLOW
Use [this glow port](https://github.com/gerekon/glow/tree/esp_llvm).
```bash
git clone -b esp_llvm --single-branch https://github.com/gerekon/glow.git
export GLOW_REPO_PATH=$PWD/glow
export GLOW_BUILD_DIR=$PWD/build_glow
mkdir -p $GLOW_BUILD_DIR
cd $GLOW_BUILD_DIR
export PATH=$LLVM_BUILD_DIR/dist/bin:$PATH
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release \
   -DLLVM_DIR=$LLVM_BUILD_DIR/dist/lib/cmake/llvm \
   -DCMAKE_C_COMPILER=$LLVM_BUILD_DIR/dist/bin/clang \
   -DCMAKE_CXX_COMPILER=$LLVM_BUILD_DIR/dist/bin/clang++ \
   $GLOW_REPO_PATH
cmake --build .
```
#### Workarounds
- You may need to install cmake 3.17.3 due to [problem with `folly` library build](https://github.com/facebook/folly/issues/1414).
- Since by default LLVM 15 treats `-Wunused-but-set-variable` warning as an error you may face the following problem when `googlebenchmark` submodule is built:
  ```
  /glow/tests/googlebenchmark/src/complexity.cc:85:10: error: variable 'sigma_gn' set but not used [-Werror,-Wunused-but-set-variable]
  double sigma_gn = 0.0;
  ```
  To solve it you can manually put `(void)sigma_gn;` somewhere in that function.

### Build GLOW AOT bundle
More details are [here](https://github.com/pytorch/glow/blob/master/docs/AOT.md).
Get LeNet and MNIST handwritten digit recognition model files (used in GLOW examples):
```bash
export MODEL_PATH=$PWD/lenet_mnist
mkdir -p $MODEL_PATH
cd $MODEL_PATH
wget http://fb-glow-assets.s3.amazonaws.com/models/lenet_mnist/init_net.pb
wget http://fb-glow-assets.s3.amazonaws.com/models/lenet_mnist/predict_net.pb
$GLOW_BUILD_DIR/bin/model-compiler -backend=CPU -model=$MODEL_PATH -emit-bundle=$PWD/build_model -model-input="data,float,[1,1,28,28]" -target=xtensa-esp-elf -mcpu=esp32s3
```
Copy files from `$PWD/build_model` to the project. Build and flash project as usual.