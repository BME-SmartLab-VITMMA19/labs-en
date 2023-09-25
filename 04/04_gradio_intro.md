
> Copyright © Bálint Gyires-Tóth, Csaba Zainkó, András Kalapos, Tamás Gábor Csapó. All Rights Reserved.
This file is protected by copyright law. The intellectual property contained herein, including but not limited to text, source code and design elements, are the exclusive property of the copyright holder identified above. Any unauthorized use, reproduction, distribution, or modification of this presentation or its contents is strictly prohibited without prior written consent from the copyright holder.
For permissions, inquiries, or licensing requests, please contact: toth.b (at) tmit.bme.hu
Unauthorized use, distribution, or reproduction of this content may result in civil and criminal penalties. Thank you for respecting the intellectual property rights of the copyright holder.


***********************************************************

# Gradio.app basics

Gradio is a solution with which you can create an interface for your machine/deep learning project in a few lines of code. You can reach it at: (https://www.gradio.app/)[https://www.gradio.app/]

# Installation
For installation, simply run

```
pip install gradio
```

# Running Gradio in Colab
After installing Gradio in Colab, run the following code.

```
import tensorflow as tf
import numpy as np
from urllib.request import urlretrieve
import gradio as gr

urlretrieve("https://gr-models.s3-us-west-2.amazonaws.com/mnist-model.h5", "mnist-model.h5")
model = tf.keras.models.load_model("mnist-model.h5")

def recognize_digit(image):
    image = image.reshape(1, -1)  
    prediction = model.predict(image).tolist()[0]
    return {str(i): prediction[i] for i in range(10)}

output_component = gr.Label(num_top_classes=5)

gr.Interface(fn=recognize_digit, 
             inputs="sketchpad", 
             outputs=output_component,
             title="Digit Sketchpad",
             description="Draw a digit between 0 and 9.").launch(share=True);
```

# Running Gradio on your own environment with Docker

First, lets define the ```Dockerfile```:

```
FROM python:3.9

ARG GRADIO_SERVER_PORT=7860
ENV GRADIO_SERVER_PORT=${GRADIO_SERVER_PORT}

EXPOSE ${GRADIO_SERVER_PORT}

WORKDIR /workspace

ADD requirements.txt main.py /workspace/

RUN pip install --upgrade pip
RUN pip install -r /workspace/requirements.txt

CMD ["python", "/workspace/main.py"]
```

and the ```requirements.txt```:

```
tensorflow==2.13.0
gradio==3.44.2
```

The content of ```main.py``` should be identical to the code, that you ran in Colab, except ```.launch()``` should not have ```share=True``` turned on, and ```server_name="0.0.0.0"``` have to be defined as follows: ```.launch(server_name="0.0.0.0")```. This ensures, that the server running in the container will inherit the IP address of the host machine.

Next, build and run the container:

```
docker build . -t gradio_example
docker run --rm -p 7869:7860 gradio_example
```

The solution will be accessible at the IP address of the machine on port 7869. Eg. 163.129.38.21:7869
