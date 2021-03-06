..
  # Copyright (c) 2018, NVIDIA CORPORATION. All rights reserved.
  #
  # Redistribution and use in source and binary forms, with or without
  # modification, are permitted provided that the following conditions
  # are met:
  #  * Redistributions of source code must retain the above copyright
  #    notice, this list of conditions and the following disclaimer.
  #  * Redistributions in binary form must reproduce the above copyright
  #    notice, this list of conditions and the following disclaimer in the
  #    documentation and/or other materials provided with the distribution.
  #  * Neither the name of NVIDIA CORPORATION nor the names of its
  #    contributors may be used to endorse or promote products derived
  #    from this software without specific prior written permission.
  #
  # THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
  # EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
  # IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
  # PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
  # CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
  # EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
  # PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
  # PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
  # OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
  # (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
  # OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

.. _section-client-libraries-and-examples:

Client Libraries and Examples
=============================

The TRTIS *client libraries* make it easy to communicate with the
TensorRT Inference Server from you C++ or Python application. Using
these libraries you can send either HTTP or GRPC requests to TRTIS to
check server status or health and to make inference requests.

A couple of example applications show how to use the client libraries
to perform image classification and to test performance:

* C++ and Python versions of *image\_client*, an example application
  that uses the C++ or Python client library to execute image
  classification models on the TensorRT Inference Server.

* Python version of *grpc\_image\_client*, an example application that
  is functionally equivalent to *image\_client* but that uses GRPC
  generated client code to communicate with TRTIS (instead of the
  client library).

* C++ version of *perf\_client*, an example application that issues a
  large number of concurrent requests to TRTIS to measure latency and
  throughput for a given model. You can use this to experiment with
  different model configuration settings for your models.

.. build-client-begin-marker-do-not-remove

.. _section-building-the-client-libraries-and-examples:

Building the Client Libraries and Examples
------------------------------------------

The provided Dockerfile can be used to build just the client libraries
and examples. Issue the following command to build the C++ client
library, C++ and Python examples, and a Python wheel file for the
Python client library::

  $ docker build -t tensorrtserver_clients --target trtserver_build --build-arg "PYVER=<ver>" --build-arg "BUILD_CLIENTS_ONLY=1" .

The -\\-build-arg setting PYVER is optional and can be used to set the
Python version that you want the Python client library built for (the
default is 3.5).

After the build completes, the easiest way to extract the built
libraries and examples from the docker image is to mount a host
directory and then copy them out from within the container::

  $ docker run -it --rm -v/tmp:/tmp/host tensorrtserver_clients
  # cp /opt/tensorrtserver/bin/image_client /tmp/host/.
  # cp /opt/tensorrtserver/bin/perf_client /tmp/host/.
  # cp /opt/tensorrtserver/bin/simple_client /tmp/host/.
  # cp /opt/tensorrtserver/pip/tensorrtserver-*.whl /tmp/host/.
  # cp /opt/tensorrtserver/lib/librequest.* /tmp/host/.

You can now access the files from /tmp on the host system. To run the
C++ examples you must install some dependencies on your host system::

  $ apt-get install curl libcurl3-dev libopencv-dev libopencv-core-dev python-pil

To run the Python examples you will need to additionally install the
client whl file and some other dependencies::

  $ apt-get install python3 python3-pip
  $ pip3 install --user --upgrade tensorrtserver-*.whl pillow

.. build-client-end-marker-do-not-remove

.. _section-image_classification_example:

Image Classification Example Application
----------------------------------------

The image classification example that uses the C++ client API is
available at `src/clients/c++/image\_client.cc
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/c%2B%2B/image_client.cc>`_. The
Python version of the image classification client is available at
`src/clients/python/image\_client.py
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/python/image_client.py>`_.

To use image\_client (or image\_client.py) you must first have a
running TRTIS that is serving one or more image classification
models. The image\_client application requires that the model have a
single image input and produce a single classification output. If you
don't have a model repository with image classification models see
:ref:`section-example-model-repository` for instructions on how to
create one.

Follow the instructions in :ref:`section-running-the-inference-server`
to launch TRTIS using the model repository. Once the server is running
you can use the image\_client application to send inference requests
to the server. You can specify a single image or a directory holding
images. Here we send a request for the resnet50_netdef model from the
:ref:`example model repository <section-example-model-repository>` for
an image from the `qa/images
<https://github.com/NVIDIA/tensorrt-inference-server/tree/master/qa/images>`_
directory::

  $ image_client -m resnet50_netdef -s INCEPTION qa/images/mug.jpg
  Request 0, batch size 1
  Image '../qa/images/mug.jpg':
      504 (COFFEE MUG) = 0.723991

The Python version of the application accepts the same command-line
arguments::

  $ src/clients/python/image_client.py -m resnet50_netdef -s INCEPTION qa/images/mug.jpg
  Request 0, batch size 1
  Image '../qa/images/mug.jpg':
      504 (COFFEE MUG) = 0.778078556061

The image\_client and image\_client.py applications use the TRTIS
client library to talk to the server. By default image\_client
instructs the client library to use HTTP protocol to talk to TRTIS,
but you can use GRPC protocol by providing the \-i flag. You must also
use the \-u flag to point at the GRPC endpoint on TRTIS::

  $ image_client -i grpc -u localhost:8001 -m resnet50_netdef -s INCEPTION qa/images/mug.jpg
  Request 0, batch size 1
  Image '../qa/images/mug.jpg':
      504 (COFFEE MUG) = 0.723991

By default the client prints the most probable classification for the
image. Use the \-c flag to see more classifications::

  $ image_client -m resnet50_netdef -s INCEPTION -c 3 qa/images/mug.jpg
  Request 0, batch size 1
  Image '../qa/images/mug.jpg':
      504 (COFFEE MUG) = 0.723991
      968 (CUP) = 0.270953
      967 (ESPRESSO) = 0.00115996

The \-b flag allows you to send a batch of images for inferencing.
The image\_client application will form the batch from the image or
images that you specified. If the batch is bigger than the number of
images then image\_client will just repeat the images to fill the
batch::

  $ image_client -m resnet50_netdef -s INCEPTION -c 3 -b 2 qa/images/mug.jpg
  Request 0, batch size 2
  Image '../qa/images/mug.jpg':
      504 (COFFEE MUG) = 0.778078556061
      968 (CUP) = 0.213262036443
      967 (ESPRESSO) = 0.00293014757335
  Image '../qa/images/mug.jpg':
      504 (COFFEE MUG) = 0.778078556061
      968 (CUP) = 0.213262036443
      967 (ESPRESSO) = 0.00293014757335

Provide a directory instead of a single image to perform inferencing
on all images in the directory::

  $ image_client -m resnet50_netdef -s INCEPTION -c 3 -b 2 qa/images
  Request 0, batch size 2
  Image '../qa/images/car.jpg':
      817 (SPORTS CAR) = 0.836187
      511 (CONVERTIBLE) = 0.0708251
      751 (RACER) = 0.0597549
  Image '../qa/images/mug.jpg':
      504 (COFFEE MUG) = 0.723991
      968 (CUP) = 0.270953
      967 (ESPRESSO) = 0.00115996
  Request 1, batch size 2
  Image '../qa/images/vulture.jpeg':
      23 (VULTURE) = 0.992326
      8 (HEN) = 0.00231854
      84 (PEACOCK) = 0.00201471
  Image '../qa/images/car.jpg':
      817 (SPORTS CAR) = 0.836187
      511 (CONVERTIBLE) = 0.0708251
      751 (RACER) = 0.0597549

The grpc\_image\_client.py application at available at
`src/clients/python/grpc\_image\_client.py
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/python/grpc_image_client.py>`_
behaves the same as the image\_client except that instead of using the
TRTIS client library it uses the GRPC generated client library to
communicate with TRTIS.

Performance Example Application
-------------------------------

The perf\_client example application located at
`src/clients/c++/perf\_client.cc
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/c%2B%2B/perf_client.cc>`_
uses the C++ client API to send concurrent requests to TRTIS to
measure latency and inferences per second under varying client loads.

To use perf\_client you must first have a running TRTIS that is
serving one or more models. The perf\_client application works with
any type of model by sending random data for all input tensors and by
reading and ignoring all output tensors. If you don't have a model
repository see :ref:`section-example-model-repository` for
instructions on how to create one.

Follow the instructions in :ref:`section-running-the-inference-server`
to launch TRTIS using the model repository.

The perf\_client application has two major modes. In the first mode
you specify how many concurrent clients you want to simulate and
perf\_client finds a stable latency and inferences/second for that
level of concurrency. Use the \-t flag to control concurrency and \-v
to see verbose output. The following example simulates four clients
continuously sending requests to TRTIS::

  $ perf_client -m resnet50_netdef -p3000 -t4 -v
  *** Measurement Settings ***
    Batch size: 1
    Measurement window: 3000 msec

  Request concurrency: 4
    Pass [1] throughput: 207 infer/sec. Avg latency: 19268 usec (std 910 usec)
    Pass [2] throughput: 206 infer/sec. Avg latency: 19362 usec (std 941 usec)
    Pass [3] throughput: 208 infer/sec. Avg latency: 19252 usec (std 841 usec)
    Client:
      Request count: 624
      Throughput: 208 infer/sec
      Avg latency: 19252 usec (standard deviation 841 usec)
      Avg HTTP time: 19224 usec (send 714 usec + response wait 18486 usec + receive 24 usec)
    Server:
      Request count: 749
      Avg request latency: 17886 usec (overhead 55 usec + queue 26 usec + compute 17805 usec)

In the second mode perf\_client will generate an inferences/second
vs. latency curve by increasing concurrency until a specific latency
limit or concurrency limit is reached. This mode is enabled by using
the \-d option and \-l to specify the latency limit and optionally the
\-c to specify a maximum concurrency limit::

  $ perf_client -m resnet50_netdef -p3000 -d -l50 -c 3
  *** Measurement Settings ***
    Batch size: 1
    Measurement window: 3000 msec
    Latency limit: 50 msec
    Concurrency limit: 3 concurrent requests

  Request concurrency: 1
    Client:
      Request count: 327
      Throughput: 109 infer/sec
      Avg latency: 9191 usec (standard deviation 822 usec)
      Avg HTTP time: 9188 usec (send/recv 1007 usec + response wait 8181 usec)
    Server:
      Request count: 391
      Avg request latency: 7661 usec (overhead 90 usec + queue 68 usec + compute 7503 usec)

  Request concurrency: 2
    Client:
      Request count: 521
      Throughput: 173 infer/sec
      Avg latency: 11523 usec (standard deviation 616 usec)
      Avg HTTP time: 11448 usec (send/recv 711 usec + response wait 10737 usec)
    Server:
      Request count: 629
      Avg request latency: 10018 usec (overhead 70 usec + queue 41 usec + compute 9907 usec)

  Request concurrency: 3
    Client:
      Request count: 580
      Throughput: 193 infer/sec
      Avg latency: 15518 usec (standard deviation 635 usec)
      Avg HTTP time: 15487 usec (send/recv 779 usec + response wait 14708 usec)
    Server:
      Request count: 697
      Avg request latency: 14083 usec (overhead 59 usec + queue 30 usec + compute 13994 usec)

  Inferences/Second vs. Client Average Batch Latency
  Concurrency: 1, 109 infer/sec, latency 9191 usec
  Concurrency: 2, 173 infer/sec, latency 11523 usec
  Concurrency: 3, 193 infer/sec, latency 15518 usec

Use the \-f flag to generate a file containing CSV output of the
results::

  $ perf_client -m resnet50_netdef -p3000 -d -l50 -c 3 -f perf.csv

You can then import the CSV file into a spreadsheet to help visualize
the latency vs inferences/second tradeoff as well as see some
components of the latency. Follow these steps:

- Open `this spreadsheet <https://docs.google.com/spreadsheets/d/1zszgmbSNHHXy0DVEU_4lrL4Md-6dUKwy_mLVmcseUrE>`_
- Make a copy from the File menu "Make a copy..."
- Open the copy
- Select the A2 cell
- From the File menu select "Import..."
- Select "Upload" and upload the file
- Select "Replace data at selected cell" and then select the "Import data" button

.. _section-client-api:

Client API
----------

The C++ client API exposes a class-based interface for querying server
and model status and for performing inference. The commented interface
is available at `src/clients/c++/request.h
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/c%2B%2B/request.h>`_
and in the API Reference.

The Python client API provides similar capabilities as the C++
API. The commented interface is available at
`src/clients/python/\_\_init\_\_.py
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/python/__init__.py>`_
and in the API Reference.

A very simple C++ example application at
`src/clients/c++/simple\_client.cc
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/c%2B%2B/simple_client.cc>`_
and a Python version at `src/clients/python/simple\_client.py
<https://github.com/NVIDIA/tensorrt-inference-server/blob/master/src/clients/python/simple_client.py>`_
demonstrate basic client API usage.

To run the the C++ version of the simple example, first build as
described in :ref:`section-building-the-client-libraries-and-examples`
and then::

  $ simple_client
  0 + 1 = 1
  0 - 1 = -1
  1 + 1 = 2
  1 - 1 = 0
  2 + 1 = 3
  2 - 1 = 1
  3 + 1 = 4
  3 - 1 = 2
  4 + 1 = 5
  4 - 1 = 3
  5 + 1 = 6
  5 - 1 = 4
  6 + 1 = 7
  6 - 1 = 5
  7 + 1 = 8
  7 - 1 = 6
  8 + 1 = 9
  8 - 1 = 7
  9 + 1 = 10
  9 - 1 = 8
  10 + 1 = 11
  10 - 1 = 9
  11 + 1 = 12
  11 - 1 = 10
  12 + 1 = 13
  12 - 1 = 11
  13 + 1 = 14
  13 - 1 = 12
  14 + 1 = 15
  14 - 1 = 13
  15 + 1 = 16
  15 - 1 = 14

To run the the Python version of the simple example, first build as
described in :ref:`section-building-the-client-libraries-and-examples`
and install the tensorrtserver whl, then::

  $ python src/clients/python/simple_client.py
