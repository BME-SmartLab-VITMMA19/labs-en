
> Copyright © Bálint Gyires-Tóth, Csaba Zainkó, András Kalapos, Tamás Gábor Csapó. All Rights Reserved.
This file is protected by copyright law. The intellectual property contained herein, including but not limited to text, source code and design elements, are the exclusive property of the copyright holder identified above. Any unauthorized use, reproduction, distribution, or modification of this presentation or its contents is strictly prohibited without prior written consent from the copyright holder.
For permissions, inquiries, or licensing requests, please contact: toth.b (at) tmit.bme.hu
Unauthorized use, distribution, or reproduction of this content may result in civil and criminal penalties. Thank you for respecting the intellectual property rights of the copyright holder.


***********************************************************

# Why we use Docker for deep learning, machine learning

- Project packaging 
  - required drivers, Linux packages
  - Python packages 
  - bundling files, scripts
- Teaching container 
  - Quick to install
  - Isolate - easily limit the folders and resources accessible by containers
  - Improve compatibility
    - if you are lucky enough to have to deal with setting up the GPU driver once or not at all
    - others can more easily reproduce our results
  - Student use of departmental GPUs
- Application container (using a trained model)
  - Easy, fast installation
  - Scalability

# Installation

Installation of Docker engine can be found here, please make sure, to select the right Linux distribution, that you are using: https://docs.docker.com/engine/install/

For GPU support, please make sure, that all the NVIDIA and CUDA drivers are installed and up to date, and install nvidia-docker: (https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)[https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html] 

## Important links

You can find Docker containers in: 
* https://hub.docker.com/
* https://ngc.nvidia.com/ 

Several examples: 

- https://hub.docker.com/_/python
- https://hub.docker.com/r/nvidia/cuda
- https://hub.docker.com/r/pytorch/pytorch
- https://hub.docker.com/r/tensorflow/tensorflow

Search for versions: the Overview tab of each page usually describes what each tag means, and the Tags tab allows you to search for all possible tags.

![](.github/image-20210909151556780.png)

- `tensorflow/tensorflow:2.3.4-gpu-jupyter` -> tensorflow framework + GPU drivers + jupyter notebook + python
- `tensorflow/tensorflow:2.3.4` ->  tensorflow framework (CPU version) + python
- `pytorch/pytorch:1.9.0-cuda10.2-cudnn7-runtime` -> pytorch framework + GPU drivers + python

Let's download a docker image containing python!

    docker pull python:latest

and run

    docker run -it python:latest


And you can try basic Docker commands here (DockerHub registration is required):
* https://labs.play-with-docker.com/

# General commands

List of containers:

```
docker ps 
docker ps -a
```

List of downloaded images: 

```
docker imges
```

**Image VS Container**: a container is an instance of the image that you can use, run as a virtual machine

Building an image from Dockerfile

```
docker build . -t {IMAGE_NAME}
```

Example run command:

```
docker run --rm --gpus all nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04 nvidia-smi
```

Example run command in interactive mode (with a prompt):

```
docker run --rm --gpus all -it nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04 bash
```

List built / downloaded docker images:

```
docker images
```

Remove a docker image: 

```
docker rmi
```

Remove a container:

```
docker rm
(if you run a docker container with --rm, then it is automatically removed, when stoped)
```
	
Remove all stopped containers:

```
docker container prune
```

# Defining a single Docker image with Dockerfile and running the container
Create a file named 'Dockerfile' with the following content:

```
FROM python:3.9.12

RUN apt-get update &&  \
	apt install -y 

ENV HOME /home/custom_user
RUN mkdir -p $HOME/project1
WORKDIR $HOME/project1
ENV CSV=${CSV:-$HOME/project1/data/data.csv}

COPY . .

RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

RUN useradd -rm -d $HOME -s /bin/bash -g root -G sudo -u 1000 custom_user
RUN echo 'custom_user:password' | chpasswd

SHELL ["/bin/bash", "-l", "-c"]

ENTRYPOINT python src/main.py -csv $CSV
```

You should put all Python package needed for your project in the requirements.txt, e.g:

```
numpy==1.25.2
pandas==2.1.0
python-dateutil==2.8.2
pytz==2023.3.post1
six==1.16.0
tzdata==2023.3
```

And the src/main.py should contain any data processing task, e.g.

```
import argparse

if __name__ == '__main__':
	parser = argparse.ArgumentParser()
	parser.add_argument(
		'-csv',
		'--csv',
		type=str,
		help='full path of the CSV file')

	args = parser.parse_args()
	print("Full path of the CSV file is:",args.csv)
```
	
When the file is saved, build the Docker image with the following command: 

```
docker build . -t smartlab:test_image
```

You can run this container with the following command, that will run the script at the ENTRYPOINT with the default $CSV environmental variable:

```
docker run --rm smartlab:test_image
```
	
You can also define a CSV file that is not built into the image, but pre-exists on the host machine:

```
docker run -e CSV=/home/custom_user/data_alternative/data.csv -v /host/folder/host_data.csv:/home/custom_user/data_alternative/data.csv --rm smartlab:test_image
```
	
If you would like to enter to the Docker container and work inside, then you have to remove the ENTRYPOINT line or modify the ENTRYPOINT command to RUN. In this case, you can run ```bash``` in the container with interactive mode (```--it```):

```
docker run -it --rm smartlab:test_image bash
```

# Defining a single Docker image with Dockerfile with GPU support, Jupter Lab and remote SSH access and running the container
In case of a GPU supported Docker image, the original source is usually based on GPU supported Docker image.
Let's see an example:

```
FROM pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime

RUN apt-get update &&  \
	apt install -y  \
	openssh-server \
	git \
	libgl1 \
	vim \
	tmux \
	byobu

ENV HOME /home/custom_user
RUN mkdir -p $HOME/project1
WORKDIR $HOME/project1
ENV CSV=${CSV:-$HOME/project1/data/data.csv}

COPY . .
COPY ssh/sshd_config /etc/ssh/sshd_config

EXPOSE 22
EXPOSE 8888

RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

RUN useradd -rm -d $HOME -s /bin/bash -g root -G sudo -u 1000 custom_user
RUN echo 'custom_user:password' | chpasswd

SHELL ["/bin/bash", "-l", "-c"]

# Start Jupyter lab with custom password
ENTRYPOINT service ssh start && jupyter-lab \
	--ip 0.0.0.0 \
	--port 8888 \
	--no-browser \
	--NotebookApp.notebook_dir='$home' \
	--ServerApp.terminado_settings="shell_command=['/bin/bash']" \
	--allow-root
```

Where requirements.txt contains the following (at least):

```
scikit-learn==1.2.2
scipy==1.8
seaborn==0.12.2
pandas==2.0.3
jupyter-client==8.1.0
jupyter-core==5.3.0
jupyter-events==0.6.3
jupyter-server==2.5.0
jupyter-server-terminals==0.4.4
jupyterlab==3.5.2
jupyterlab-pygments==0.2.2
```

And ssh/sshd_config contains the following (at least):

```
Port 22
PermitRootLogin no
AllowUsers custom_user
```
	
When the file is saved, build the Docker image with the following command: 

```
docker build . -t smartlab:test_image_gpu
```

And then run the Docker image:

```
docker run --gpus all -v ${pwd}:/home/custom_user -p 2299:22 -p 8899:8888 --rm --name test_container_gpu test_image_gpu
```

# Define and run a single Docker image with Docker compose with GPU support

If you would like to set up the container with Docker compose, let's define a ```docker-compose.yml``` file, as follows:

```
services:
  srserver:
	build:
	  context: ./
	  dockerfile: Dockerfile
	image: smartlab:test_image_gpu
	container_name: test_container_gpu
	volumes:
	  - .:/home/custom_user/
	ports:
	  - "8899:8888"
	  - "2299:22"
	restart: unless-stopped
	deploy:
	  resources:
		reservations:
		  devices:
			- driver: "nvidia"
			  capabilities: [ "gpu" ]
			  count: 1
```

You can build and run the solution as follows:

```
docker-compose build
docker-compose up
```

