![Logo](../figures/icons/onnx.png){ align=right width="130"}

# Deployment of Machine Learning Models

---

In one of the [previous modules](../s7_deployment/apis.md) you learned about how to use
[FastAPI](https://fastapi.tiangolo.com/) to create an API to interact with your machine learning models. FastAPI is a
great framework, but it is a general framework meaning that it was not developed with machine learning applications in
mind. This means that there are features which you may consider to be missing when considering running large scale
machine learning models:

* Dynamic-batching: if you have a large number of requests coming in, you may want to process them in batches to
    reduce the overhead of loading the model and running the inference. This is especially true if you are running your
    model on a GPU, where the overhead of loading the model is significant.

* Async inference: FastAPi does support async requests but not no way to call the model asynchronously. This means that
    if you have a large number of requests coming in, you will have to wait for the model to finish processing (because
    the model is not async) before you can start processing the next request.

* Native GPU support: you can definitely run part of your application in FastAPI if you want to. But again it was not
    build with machine learning in mind, so you will have to do some extra work to get it to work.

It should come as no surprise that multiple frameworks have therefore sprung up that better supports deployment of
machine learning algorithms:

* [Cortex](https://github.com/cortexlabs/cortex)

* [Bento ML](https://github.com/bentoml/bentoml)

* [Ray Serve](https://docs.ray.io/en/master/serve/)

* [Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html)

* [OpenVINO](https://docs.openvino.ai/2024/index.html)

* [Seldon-core](https://docs.seldon.io/projects/seldon-core/en/latest/)

* [Torchserve](https://pytorch.org/serve/)

* [Tensorflow serve](https://github.com/tensorflow/serving)

The first 6 frameworks are backend agnostic, meaning that they are intended to work with whatever computational backend
you model is implemented in (Tensorflow vs PyTorch vs Jax), whereas the last two are backend specific to respective
Pytorch and Tensorflow. In this module we are going to look at two of the frameworks, namely `Torchserve` because we
have developed Pytorch applications i nthis course and `Triton Inference Server` because it is a very popular framework
for deploying models on Nvidia GPUs (but we can still use it on CPU). We recommend that yo

But before we dive into these frameworks, we are going to look at a general way to package our machine learning models
that should work with any of the above frameworks.

## Model Packaging

Whenever we want to serve an machine learning model, we in general need 3 things:

* The computational graph of the model, e.g. how to pass data through the model to get a prediction.
* The weights of the model, e.g. the parameters that the model has learned during training.
* A computational backend that can run the model

In the previous module on [Docker](../s3_reproducibility/docker.md) we learned how to package all of these things into
a container. This is a great way to package a model, but it is not the only way. The core assumption we currently have
made is that the computational backend is the same as the one we trained the model on. However, this does not need to
be the case. As long as we can export our model and weights to a common format, we can run the model on any backend
that supports this format.

This is exactly what the [Open Neural Network Exchange (ONNX)](https://onnx.ai/) is designed to do. ONNX is a
standardized format for creating and sharing machine learning models. It defines an extensible computation graph model,
as well as definitions of built-in operators and standard data types. The idea behind ONNX is that a model trained with
a specific framework on a specific device, lets say Pytorch on your local computer, can be exported and run with an
entirely different framework and hardware easily. Learning how to export your models to ONNX is therefore a great way
to increase the longivity of your models and not being locked into a specific framework for serving your models.

<figure markdown>
![Image](../figures/onnx.png){ width="1000" }
<figcaption>
The ONNX format is designed to bridge the gap between development and deployment of machine learning models, by making
it easy to export models between different frameworks and hardware. For example Pytorch is in general considered
an developer friendly framework, however it has historically been slow to run inference with compared to a framework.
<a href="https://medium.com/trueface-ai/two-benefits-of-the-onnx-library-for-ml-models-4b3e417df52e"> Image credit </a>
</figcaption>
</figure>

### ❔ Exercises

1. Start by installing ONNX, ONNX runtime and ONNX script. This can be done by running the following command

    ```bash
    pip install onnx onnxruntime onnxscript
    ```

    the first package contains the core ONNX framework, the second package contains the runtime for running ONNX models
    and the third package contains a new experimental package that is designed to make it easier to export models to
    ONNX.

2. Let's start out with converting a model to ONNX. The following code snippets shows how to export a Pytorch model to
    ONNX.

    === "Pytorch => 2.0"

        ```python
        import torch
        import torchvision

        model = torchvision.models.resnet18(weights=None)
        model.eval()

        dummy_input = torch.randn(1, 3, 224, 224)
        onnx_model = torch.onnx.dynamo_export(
            model=model,
            model_args=(dummy_input,),
            export_options=torch.onnx.ExportOptions(dynamic_shapes=True),
        )
        onnx_model.save("resnet18.onnx")
        ```

    === "Pytorch < 2.0 or Windows"

        ```python
        import torch
        import torchvision

        model = torchvision.models.resnet18(weights=None)
        model.eval()

        dummy_input = torch.randn(1, 3, 224, 224)
        torch.onnx.export(
            model=model,
            args=(dummy_input,),
            f="resnet18.onnx",
            input_names=["input"],
            output_names=["output"],
            dynamic_axes={"input": {0: "batch_size"}, "output": {0: "batch_size"}}
        )
        ```

    === "Pytorch-lightning"

        ```python
        import torch
        import torchvision
        import pytorch_lightning as pl
        import onnx
        import onnxruntime

        class LitModel(pl.LightningModule):
            def __init__(self):
                super().__init__()
                self.model = torchvision.models.resnet18(pretrained=True)
                self.model.eval()

            def forward(self, x):
                return self.model(x)

        model = LitModel()
        model.eval()

        dummy_input = torch.randn(1, 3, 224, 224)
        model.to_onnx(
            file_path="resnet18.onnx",
            input_sample=dummy_input,
            input_names=["input"],
            output_names=["output"],
            dynamic_axes={"input": {0: "batch_size"}, "output": {0: "batch_size"}}
        )
        ```

    Export a model of your own choice to ONNX or just try to export the `resnet18` model as shown in the examples above,
    and confirm that the model was exported by checking that the file exists. Can you figure out what is meant by
    `dynamic_axes`?

    ??? success "Solution"

        The `dynamic_axes` argument is used to specify which axes of the input tensor that should be considered dynamic.
        This is useful when the model can accept inputs of different sizes, e.g. when the model is used in a dynamic
        batching scenario. In the example above we have specified that the first axis of the input tensor should be
        considered dynamic, meaning that the model can accept inputs of different batch sizes. While it may be tempting
        to specify all axes as dynamic, however this can lead to slower inference times, because the ONNX runtime will
        not be able to optimize the computational graph as well.

3. Check that the model was correctly exported by loading it using the `onnx` package and afterwards check the graph
    of model using the following code:

    ```python
    import onnx
    model = onnx.load("resnet18.onnx")
    onnx.checker.check_model(model)
    print(onnx.helper.printable_graph(model.graph))
    ```

4. To get a better understanding of what is actually exported, lets try to visualize the computational graph of the
    model. This can be done using the open-source tool [netron](https://github.com/lutzroeder/netron). You can either
    try it out directly in [webbrowser](https://netron.app/) or you can install it locally using `pip install netron`
    and then run it using `netron resnet18.onnx`. Can you figure out what method of the model is exported to ONNX?

    ??? success "Solution"

        When a Pytorch model is exported to ONNX, it is only the `forward` method of the model that is exported. This
        means that it is the only method we have access to when we load the model later. Therefore, make sure that the
        `forward` method of your model is implemented in a way that it can be used for inference.

5. After converting a model to ONNX format we can use the [ONNX Runtime](https://onnxruntime.ai/docs/) to run it.
    The benefit of this is that ONNX Runtime is able to optimize the computational graph of the model, which can lead
    to faster inference times. Lets try to look into that.

    1. Figure out how to run a model using the ONNX Runtime. Relevant
        [documentation](https://onnxruntime.ai/docs/get-started/with-python.html).

        ??? success "Solution"

            To use the ONNX runtime to run a model, we first need to start a inference session, then extract input
            output names of our model and finally run the model. The following code snippet shows how to do this.

            ```python
            import onnxruntime as rt
            ort_session = rt.InferenceSession("<path-to-model>")
            input_names = [i.name for i in ort_session.get_inputs()]
            output_names = [i.name for i in ort_session.get_outputs()]
            batch = {input_names[0]: np.random.randn(1, 3, 224, 224).astype(np.float32)}
            out = ort_session.run(output_names, batch)
            ```

    2. Let's experiment with performance of ONNX vs. Pytorch. Implement a benchmark that measures the time it takes to
        run a model using Pytorch and ONNX. Bonus points if you test for multiple input sizes. To get you started we
        have implemented a timing decorator that you can use to measure the time it takes to run a function.

        ```python
        from statistics import mean, stdev
        import time
        def timing_decorator(func, function_repeat: int = 10, timing_repeat: int = 5):
            """ Decorator that times the execution of a function. """
            def wrapper(*args, **kwargs):
                timing_results = []
                for _ in range(timing_repeat):
                    start_time = time.time()
                    for _ in range(function_repeat):
                        result = func(*args, **kwargs)
                    end_time = time.time()
                    elapsed_time = end_time - start_time
                    timing_results.append(elapsed_time)
                print(f"Avg +- Stddev: {mean(timing_results):0.3f} +- {stdev(timing_results):0.3f} seconds")
                return result
            return wrapper
        ```

        ??? success "Solution"

            ```python linenums="1" title="onnx_benchmark.py"
            --8<-- "s10_extra/exercise_files/onnx_benchmark.py"
            ```

    3. To get a better understanding of why running the model using the ONNX runtime is usually faster lets try to see
        what happens to the computational graph. By default the ONNX Runtime will apply these optimization in *online*
        mode, meaning that the optimizations are applied when the model is loaded. However, it is also possible to apply
        the optimizations in *offline* mode, such that the optimized model is saved to disk. Below is an example of how
        to do this.

        ```python
        import onnxruntime as rt
        sess_options = rt.SessionOptions()

        # Set graph optimization level
        sess_options.graph_optimization_level = rt.GraphOptimizationLevel.ORT_ENABLE_EXTENDED

        # To enable model serialization after graph optimization set this
        sess_options.optimized_model_filepath = "optimized_model.onnx>"

        session = rt.InferenceSession("<model_path>", sess_options)
        ```

        Try to apply the optimizations in offline mode and use `netron` to visualize both the original and optimized
        model side by side. Can you see any differences?

        ??? success "Solution"

            You should hopefully see that the optimized model consist of fewer nodes and edges than the original model.
            These nodes are often called fused nodes, because they are the result of multiple nodes being fused
            together. In the image below we have visualized the first part of the computational graph of a resnet18
            model, before and after optimization.

            <figure markdown>
            ![Image](../figures/onnx_optimization.png){ width="600" }
            </figure>

6. As mentioned in the introduction, ONNX is able to run on many different types of hardware and execution engine.
    You can check all providers and all the available providers by running the following code

    ```python
    import onnxruntime
    print(onnxruntime.get_all_providers())
    print(onnxruntime.get_available_providers())
    ```

    Can you figure out how to set which provide the ONNX runtime should use?

    ??? success "Solution"

        The provider that the ONNX runtime should use can be set by passing the `providers` argument to the
        `InferenceSession` class. A list should be provided, which prioritizes the providers in the order they are
        listed.

        ```python
        import onnxruntime as rt
        provider_list = ['CUDAExecutionProvider', 'CPUExecutionProvider']
        ort_session = rt.InferenceSession("<path-to-model>", providers=provider_list)
        ```

        In this case we will prefer CUDA Execution Provider over CPU Execution Provider if both are available.

7. As you have probably realised in the exercises [on docker](../s3_reproducibility/docker.md), it can take a long time
    to build the kind of containers we are working with and they can be quite large. There is a reason for this and that
    is that Pytorch is a very large framework with a lot of dependencies. ONNX on the other hand is a much smaller
    framework. This kind of makes sense, because Pytorch is a framework that primarily was designed for developing e.g.
    training models, while ONNX is a framework that is designed for serving models. Let's try to quantify this.

    1. Construct a dockerfile that builds a docker image with Pytorch as a dependency. The dockerfile does actually
        not need to run anything. Repeat the same process for the ONNX runtime. Bonus point for developing a docker
        image that takes a [build arg](https://docs.docker.com/build/guide/build-args/) at build time that specifies
        if the image should be built with CUDA support or not.

        ??? success "Solution"

            The dockerfile for the Pytorch image could look something like this

            ```dockerfile linenums="1" title="inference_pytorch.dockerfile"
            --8<-- "s10_extra/exercise_files/inference_pytorch.dockerfile"
            ```

            and the dockerfile for the ONNX image could look something like this

            ```dockerfile linenums="1" title="inference_onnx.dockerfile"
            --8<-- "s10_extra/exercise_files/inference_onnx.dockerfile"
            ```

    2. Build both containers and measure the time it takes to build them. How much faster is it to build the ONNX
        container compared to the Pytorch container?

        ??? success "Solution"

            On unix/linux you can use the [time](https://linuxize.com/post/linux-time-command/) command to measure
            the time it takes to build the containers. Building both images, with and without CUDA support, can be done
            with the following commands

            ```bash
            time docker build . -t pytorch_inference_cuda:latest -f inference_pytorch.dockerfile \
                --no-cache --build-arg CUDA=true
            time docker build . -t pytorch_inference:latest -f inference_pytorch.dockerfile \
                --no-cache --build-arg CUDA=
            time docker build . -t onnx_inference_cuda:latest -f inference_onnx.dockerfile \
                --no-cache --build-arg CUDA=true
            time docker build . -t onnx_inference:latest -f inference_onnx.dockerfile \
                --no-cache --build-arg CUDA=
            ```

            the `--no-cache` flag is used to ensure that the build process is not cached and ensure a fair comparison.
            On my laptop this respectively took `5m1s`, `1m4s`, `0m4s`, `0m50s` meaning that the ONNX
            container was respectively 7x (with CUDA) and 1.28x (no CUDA) faster to build than the Pytorch container.

    3. Find out the size of the two docker images. It can be done in the terminal by running the `docker images`
        command. How much smaller is the ONNX model compared to the Pytorch model?

        ??? success "Solution"

            As of writing the docker image containing the Pytorch framework was 5.54GB (with CUDA) and 1.25GB (no CUDA).
            In comparison the ONNX image was 647MB (with CUDA) and 647MB (no CUDA). This means that the ONNX image is
            respectively 8.5x (with CUDA) and 1.94x (no CUDA) smaller than the Pytorch image.

8. (Optional) Assuming you have completed the module on [FastAPI](../s7_deployment/apis.md) try creating a small
    FastAPI application that serves a model using the ONNX runtime.

    ??? success "Solution"

        Here is a simple example of how to create a FastAPI application that serves a model using the ONNX runtime.

        ```python linenums="1" title="onnx_fastapi.py"
        --8<-- "s10_extra/exercise_files/onnx_fastapi.py"
        ```

This completes the exercises on the ONNX format. Do note that one limitation of the ONNX format is that is is based on
[ProtoBuf](https://protobuf.dev/), which is a binary format. A protobuf file can have a maximum size of 2GB, which means
that the `.onnx` format is not enough for very large models. However, through the use of
[external data](https://onnxruntime.ai/docs/tutorials/web/large-models.html) it is possible to circumvent this
limitation.

## Torchserve

[Torchserve](https://pytorch.org/serve/) is the native model serving framework for Pytorch. It is designed to be a
lightweight, flexible and easy to use framework for serving Pytorch models. The overall architecture of `torchserve`
can be seen in the image below. While you do not have to understand every part of the architecture, overall note that:

* Torchserve consist of a backend and a *frontend*. When a request is send through the inference API it is first send to
    the frontend. The frontend can consist of multiple endpoints, serving multiple models. The frontends main task
    is to route the request to the correct backend.
* The specific *backend* that the request is routed to is determined by information in the request. The backend is
    responsible for loading the model, running the inference and returning the result to the frontend.
* The backend can run multiple models at the same time. The models are loaded from a common *model store*, which is
    a directory on the server where the models are stored.
* Finally, torchserve also consist of a management API, which can be used to manage the server, e.g. to start and stop
    the server, to load and unload models and to get information about the server without having to shut it down.

<figure markdown>
![Image](../figures/torchserve.png){ width="1000" }
<figcaption>
<a href="https://pytorch.org/serve/"> Image credit </a>
</figcaption>
</figure>

### ❔ Exercises

1. You can install `torchserve` and its additional dependencies locally with the following command

    ```bash
    pip install torchserve torch-model-archiver torch-workflow-archiver
    ```

    however, `torchserve` requires you to have Java installed on your system. For this reason we are going to go through
    the exercises using docker, but you can try to install it locally if you want to (we will need the
    `torch-model-archiver` in both cases). Else lets start by pulling the relevant docker image:

    ```bash
    docker pull pytorch/torchserve:latest
    ```

2. Create a folder called `model-store`, where we are going to store our models. If you choose to install `torchserve`
    locally you can check if the installation was successful by running the following command

    ```bash
    torchserve --model-store model-store
    ```

    else try to run the docker image with the
    [following command](https://github.com/pytorch/serve/blob/master/docker/README.md)

    ```bash
    docker run --rm -it \
        -p 127.0.0.1:8080:8080 \
        -p 127.0.0.1:8081:8081 \
        -p 127.0.0.1:8082:8082 \
        -p 127.0.0.1:7070:7070 \
        -p 127.0.0.1:7071:7071 \
        pytorch/torchserve:latest
    ```

    TorchServe's Dockerfile configures ports 8080, 8081 , 8082, 7070 and 7071 to be exposed to the host by default. In
    both cases you should be able to access the [admin panel](https://pytorch.org/serve/management_api.html) (which is
    served on port `8081`) by going to `http://localhost:8081/models`. Since we did not load any models yet, the admin
    panel should be empty.

3. We need to write a custom handler that tells `torchserve` how to use our model for inference. Take a look at this
    [documentation](https://pytorch.org/serve/custom_service.html) and write a custom handler in a file called
    `onnx_handler.py` that can be used to serve the `resnet18` model we exported to ONNX in the previous exercises. Here
    are some starting code to get you started.

    ??? example "Starter code"

        ```python linenums="1" title="onnx_handler.py"
        --8<-- "s10_extra/exercise_files/onnx_handler_fillout.py"
        ```

    ??? success "Solution"

        ```python linenums="1" title="onnx_handler.py"
        --8<-- "s10_extra/exercise_files/onnx_handler.py"
        ```

4. We are now going to reuse the `resnet18.onnx` file we created in the previous set of exercises. For `torchserve` to
    work with the model we need to convert it to a `.mar` file. This can be done using the `torch-model-archiver`
    package. The following command shows how to do this

    ```bash
    torch-model-archiver --model-name resnet18 --version 1.0 \
        --model-file resnet18.onnx --serialized-file resnet18.mar --handler onnx_handler.py
    ```

5. We can now start the `torchserve` server using the following command

    ```bash
    docker run --rm -it \
        -p 8080:8080 -p 8081:8081 \
        -v $(pwd)/model_store:/home/model-server/model-store \
        pytorch/torchserve:latest
    ```

    The server should now be running and you can access the admin panel by going to `http://localhost:8081`. You can
    also test the model by sending a request to `http://localhost:8080/predictions/resnet18` with a image as input.

6. Torchserve is able to serve multiple models. First convert a `resnet34` model to ONNX and then use the
    `torch-model-archiver` to convert it to a `.mar` file (you should be able to use the same handler). Put the
    `resnet34.mar` file in the `model-store` folder and restart the server. You should now be able to access the
    `resnet34` model in the admin panel.



## Triton Inference Server

> Triton Inference Server is an open source inference serving software that streamlines AI inferencing.

At the core of the triton inference server is the concept of a model repository. This is a directory where you place
your models and a configuration file that tells the server how to load the model. The server supports many different
model formats:




If you have completed the previous [module on ONNX](onnx.md) the you can just place the `.onnx` file in the model
repository as triton-inference server also supports ONNX models.

```txt
<model-repository-path>/
    <model-name>/
        config.pbtxt
        <version-number>/
            model.onnx
        <version-number>/
            model.onnx
    <model-name>/
        ...
```

Regardless of format, the overall structure is the same: the inference server can serve multiple models at the same time
and therefore each model has its own directory. Inside each model directory there is a `config.pbtxt` file that tells the
server how to load the model. The `config.pbtxt` file is a protobuf file that contains the following information:

* `name`: the name of the model
* `platform`: the platform that the model is running on (e.g. `onnxruntime_onnx`)
* `max_batch_size`: the maximum batch size that the model can handle
* `input`: the input tensor names and shapes
* `output`: the output tensor names and shapes
* `version_policy`: the version policy for the model (e.g. `latest`)
* `dynamic_batching`: whether dynamic batching is enabled
* `optimization`: whether the model is optimized for inference

### ❔ Exercises

1. define `config.pbtxt`

    ```txt
    name: "text_detection"
    backend: "onnxruntime"
    max_batch_size : 256
    input [
    {
        name: "input_images:0"
        data_type: TYPE_FP32
        dims: [ -1, -1, -1, 3 ]
    }
    ]
    output [
    {
        name: "feature_fusion/Conv_7/Sigmoid:0"
        data_type: TYPE_FP32
        dims: [ -1, -1, -1, 1 ]
    }
    ]
    output [
    {
        name: "feature_fusion/concat_3:0"
        data_type: TYPE_FP32
        dims: [ -1, -1, -1, 5 ]
    }
    ]
    ```

1. Triton inference server can be installed locally, however we are again going to recommend that you just use docker to run the server. Start by pulling the relevant docker image

    ```bash
    nvcr.io/nvidia/tritonserver:24.06-trtllm-python-py3  #(1)!
    ```

    1. :man_raising_hand: `nvcr` is the nvidia container registry which can be found 
        [here](https://catalog.ngc.nvidia.com/containers). The specific container can be found
        [here](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tritonserver).

2. To check that everything works as it should lets try to run the inference server using the following command

    !!! "CPU"

        ```bash
        docker run -it --rm --net=host -v ${PWD}:/workspace/ nvcr.io/nvidia/tritonserver:24.06-trtllm-python-py3 bash
        ```

    !!! "GPU"

        ```bash
        docker run -it --rm --net=host -v ${PWD}:/workspace/ nvcr.io/nvidia/tritonserver:24.06-trtllm-python-py3-sdk bash
    ``  ```

1. docker run -it --rm --net=host -v ${PWD}:/workspace/ nvcr.io/nvidia/tritonserver:<yy.mm>-py3-sdk bash

2. You should be able to call it as any other requests but lets try out the client that comes with the triton server

    ```bash
    pip install tritonclient
    ```
    ```bash
    triton-inference-server-client -m text_detection -i /workspace/input_images.npy -o /workspace/output_images.npy
    ```

    ```python
    import tritonclient.grpc as grpcclient
    import tritonclient.grpc.model_config_pb2 as mc
    import tritonclient.http as httpclient

    # Create a GRPC client
    triton_client = grpcclient.InferenceServerClient(url="localhost:8001", verbose=True)
    ```

3. (Optional) Out of the box Triton Inference server is a really fast inference server. However, we can improve performance
    even more by taking advantage of the 
    [Triton Performance Analyzer](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/client/src/c%2B%2B/perf_analyzer/README.html)
    to analyze different configs of our server, to find the best configuration for our model.

4. (Optional) Triton Inference server support many different inference engines. We have here used the onnx inference
    engine because of ONNX portability. However, if we really want to focus on performance there is a inference engine
    that in general works better with Nvidia GPUs, namely [TensorRT](https://github.com/NVIDIA/TensorRT). Try to convert
    your ONNX model into the TensorRT format and see if you can get better performance. You can use the `trtexec` tool
    that comes with TensorRT to convert the model.

    ```bash
    trtexec --onnx=resnet18.onnx --saveEngine=resnet18.engine
    ```

    Here is a [link](https://docs.nvidia.com/deeplearning/tensorrt/quick-start-guide/index.html) to a quick start guide
    for TensorRT. Try executing the model using the TensorRT engine and compare the performance to the ONNX engine.

    ??? success "Solution"

        On my personal laptop the ONNX engine was able to process 1000 requests in 1.5 seconds, while the TensorRT engine was
        able to process the same number of requests in 1.2 seconds. This is a 20% improvement in performance.



## 🧠 Knowledge check

1. How would you export a `scikit-learn` model to ONNX? What method is exported when you export `scikit-learn` model to
    ONNX?

    ??? success "Solution"

        It is possible to export a `scikit-learn` model to ONNX using the `sklearn-onnx` package. The following code
        snippet shows how to export a `scikit-learn` model to ONNX.

        ```python
        from sklearn.ensemble import RandomForestClassifier
        from skl2onnx import to_onnx
        model = RandomForestClassifier(n_estimators=2)
        dummy_input = np.random.randn(1, 4)
        onx = to_onnx(model, dummy_input)
        with open("model.onnx", "wb") as f:
            f.write(onx.SerializeToString())
        ```

        The method that is exported when you export a `scikit-learn` model to ONNX is the `predict` method.

2. In your own words, describe what the concept of *computational graph* means?

    ??? success "Solution"

        A computational graph is a way to represent the mathematical operations that are performed in a model. It is
        essentially a graph where the nodes are the operations and the edges are the data that is passed between them.
        The computational graph normally represents the forward pass of the model and is the reason that we can easily
        backpropagate through the model to train it, because the graph contains all the necessary information to
        calculate the gradients of the model.

3. Explain why fusing operations together in the computational graph often leads to better performance?

    ??? success "Solution"

        Each time we want to do a computation, the data needs to be loaded from memory into the CPU/GPU. This is a
        slow process and the more operations we have, the more times we need to load the data. By fusing operations
        together, we can reduce the number of times we need to load the data, because we can do multiple operations
        on the same data before we need to load new data.

This ends the module on tools specifically designed for serving machine learning models.