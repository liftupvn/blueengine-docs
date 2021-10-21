# BlueEngine Document

Welcome to the BlueEngine document. Whether you are a back-end developer, or an AI engineer, in order to use the BlueEngine ecosystem, there are three main parts you need to care about:

## Key features

The **BlueEngine SDK** is what an AI engineer will use to transform an AI model, or any kind of processing code in general, to an **engine** that can be integrated into the **BlueEdge system**. Using the BlueEngine ecosystem to build engines give you several benefits:

- Build an engine is as simple as writing Python functions.
- Easily create waterfall style multi-step processing pipeline, each step run in its own [Process](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.Process).
- Auto scale the Processes to handle multiple concurent inputs. This is the key feature to raise the number of CCU that the big system can handle, with minimum cost.
- Easily send and receive any type of data to and from other engines or services, from JSON to image and video streams.
- Create Dynamic Acyclic Graphs (DAGs) that can combine multiple engines together to form a complicated, high performance processing pipeline. DAGs can be created on web UI or with Python scripts (*TODO*)
- Smart auto scaling mechanism, scale by number of incomming requests. Allow scale-to-zero to minimize operation cost.
- Support webhooks.
- Deploy to cloud, on-prem, or hybrid systems.
- Ready-to-use monitoring system.

## Roadmap

- [ ] Web UI to create DAGs
- [ ] Python SDK to create DAGs