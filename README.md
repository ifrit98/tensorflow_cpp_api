# tensorflow_cpp_api
Installation instructions for accessing the tensorflow C and C++ APIs outside of bazel


Tensorflow deleted their cc api guide, so after struggling myself with this
I thought I would share a (somewhat) friendly way to build the C++ api and include all
necessary headers to use gcc/g++/Cmake instead of bazel for compilation without having to 
put all of your project source code in the tensorflow source tree.  Yuck.


Unfortunately, as of January 27, 2020, the c_api does not yet support tensorflow 2, so we will
build against tensorflow==1.15.


#### Note: I will be using python3.6 here.  Update for your versioning accordingly.


### Fresh venv for safety
`virtualenv --no-site-packages --python=/usr/bin/python3 venv`
`. venv/bin/activate`


### Download and install tensorflow pip package to make sure deps are in site-packages:
`python -m pip install --ignore-installed tensorflow`



### Temporarily set $HOME to your localdrive so bazel cache doesn't slow down network
`export HOME=/home/localdrive`
`echo $HOME`
`cd $HOME`


### Download and move tensorflow c api binary (libtensorflow.so) to a local path
##### Can be found here if link is broken: https://www.tensorflow.org/install/lang_c#download
`wget https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-linux-x86_64-1.15.0.tar.gz`


### This tar contains two folders, lib and include, which we will use to store .so binary and .h header files, respectively
##### Ideally this will go in your /usr/local, but if you don't have sudo, put it in ~/.local, which is what I do
`mkdir ~/.local/tensorflow`
`tar -C ~/.local/tensorflow -xzf libtensorflow-gpu-linux-x86_64-1.15.0.tar.gz`
`delete libtensorflow-gpu-linux-x86_64-1.15.0.tar.gz`


### If extracted to a system dir, like /usr/local, configure linker with ldconfig
`sudo ldconfig`


### Add C api library paths to ~/.bashrc
`export LIBRARY_PATH=$LIBRARY_PATH:~/.local/tensorflow/lib`
`export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/.local/tensorflow/lib`

### Add C include path
`export CPATH=/mnt/ipahome/georgej/.local/tensorflow/include`


### Confirm proper installation by creating hello_tf.c, compiling with gcc and executing
```{c}
#include <stdio.h>
#include <tensorflow/c/c_api.h>

int main() {
  printf("Hello from TensorFlow C library version %s\n", TF_Version());
  return 0;
}
```

`gcc hello_tf.c -ltensorflow -o hello_tf`
`./hello_tf`

#### If the program doesn't build make sure that gcc can access the TF C library:
##### Explicitly pass library location to compiler:
`gcc -I/usr/local/include -L/usr/local/lib hello_tf.c -ltensorflow -o hello_tf`
##### Or if working in ~/.local
`gcc -I~/.local/tensorflow/include -L~/.local/lib hello_tf.c -ltensorflow -o hello_tf`

### Success, the C api is now installed!



### clone tensorflow repo and checkout appropriate branch (1.15 in this case)
`git clone https://github.com/tensorflow/tensorflow.git`
`cd tensorflow`
`git checkout r1.15`


### Make sure bazel-1.2.1 is installed and aliased over any other bazel installation
https://docs.bazel.build/versions/master/install-ubuntu.html

#### Add an alias to .bashrc to mask newer versions of bazel
`export alias bazel=bazel-1.2.1`

# Configure the bazel build appropriately for your system
`./configure`
...

### Build tensorflow  C++ static library binary (libtensorflow_cc.so)
`bazel build --config=monolithic --config=v1 --config=opt --config=cuda //tensorflow:libtensorflow_cc.so`

### Add headers and binaries to relevant paths
##### Keep source where it was built so not to break .so symlinks or waste network bandwidth
`export CPATH=$CPATH:/home/localdrive/tensorflow` # where the C++ headers live


### Add where the C++ binaries live to your $LIBRARY_PATH(s)
`export LIBRARY_PATH=$LIBRARY_PATH:/home/localdrive/tensorflow/bazel-bin/tensorflow`
`export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/localdrive/tensorflow/bazel-bin/tensorflow`


### Add C++ include paths
`export CPATH=$CPATH:/home/localdrive/tensorflow`
`export CPATH=$CPATH:/home/localdrive/tensorflow/bazel-bin`

### Add one final C++ include path from the python .whl in your venv (e.g. for Eigen issue) 
Re: https://github.com/tensorflow/tensorflow/issues/4680 and https://github.com/tensorflow/tensorflow/issues/13705
`export CPATH=$CPATH:~/venv/lib/python3.6/site-packages/tensorflow_core/include`


### Compile/run this minimal code example by creating hello_cpp.cc to test the C++ api
```{c++}

#include <iostream>

#include "tensorflow/cc/client/client_session.h"
#include "tensorflow/cc/ops/standard_ops.h"
#include "tensorflow/core/framework/tensor.h"

using namespace std;


int main() {
  using namespace tensorflow;
  using namespace tensorflow::ops;
  Scope root = Scope::NewRootScope();
  // Matrix A = [3 2; -1 0]
  auto A = Const(root, { {3.f, 2.f}, {-1.f, 0.f} });
  // Vector b = [3 5]
  auto b = Const(root, { {3.f, 5.f} });
  // v = Ab^T
  auto v = MatMul(root.WithOpName("v"), A, b, MatMul::TransposeB(true));
  std::vector<Tensor> outputs;
  ClientSession session(root);
  // Run and fetch v
  TF_CHECK_OK(session.Run({v}, &outputs));
  // Expect outputs[0] == [19; -3]
  LOG(INFO) << outputs[0].matrix<float>();

  cout << "Successfully completed!" << endl;


  return 0;
}
```


#### To compile and run:
`g++ hello_cpp.cc -ltensorflow_cc -o hello_cpp`
`./hello_cpp`


# Congratulations!  You now have access to the tensorflow C and C++ API!

##### In general, to compile:
For C:
gcc file.c -ltensorflow ... -o file

For C++:
g++ file.cc -ltensorflow_cc ... -o file



### References: 
https://www.tensorflow.org/install/lang_c
https://stackoverflow.com/questions/33620794/how-to-build-and-use-google-tensorflow-c-api
