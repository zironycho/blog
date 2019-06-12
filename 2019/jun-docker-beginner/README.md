# Docker usage

>For the beginner





## Docker architecture

> docker architecture

![](https://docs.docker.com/engine/images/architecture.svg)











## Docker run

> docker run을 통해 컨테이너를 실행

### basic run command

```
$ docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```



### basic options

```
Options:
  -d, --detach                         Run container in background and print container ID
  -e, --env list                       Set environment variables
  -i, --interactive                    Keep STDIN open even if not attached
  -p, --publish list                   Publish a container's port(s) to the host
      --rm                             Automatically remove the container when it exits
  -t, --tty                            Allocate a pseudo-TTY
  -v, --volume list                    Bind mount a volume
  -w, --workdir string                 Working directory inside the container
```



### examples

#### simple run 

```bash
$ docker run -it \
	{{base image}} \
	python train.py
```

#### background (detached)

```bash
$ docker run -dt \
	zironycho/pytorch:1.1.0-slim \
	python train.py
```

#### volume mount & working directory

```bash
$ docker run -it \
	-v $(pwd):/workspace \
	-w /workspace \
	zironycho/pytorch:1.1.0-slim \
	python train.py
```

#### remove container when it exits

```bash
$ docker run -it \
	--rm \
	zironycho/pytorch:1.1.0-slim \
	python train.py
```

#### publish port

```bash
$ docker run -it \
	-p 9999:8888 \
	zironycho/pytorch-jupyter:1.1.0-slim \
	jupyter notebook  --ip 0.0.0.0 --port 8888 --no-browser --allow-root
```

* ip에 주의

#### env

```bash
$ docker run -it \
	-e MAX_EPOCH=100 \
	-v $(pwd)/tacotron:/workspace \
	-v /database:/database/data \
	-w /workspace \
	zironycho/pytorch:1.1.0-slim \
	python train.py
```



### container exit이후

* volume으로  mount하지 않는 이상 날아감





## ngc(nvidia gpu cloud)

> registry for nvidia optimized image

* nvcr.io/nvidia/pytorch:19.05-py3
* ...









## Docker exec, logs, ps, stop, images

> 그 밖의 자주 사용하는 commands



### docker exec를 컨테이너에 사용하여 접속

```bash
$ docker exec -it {{container_id}} bash
```

### docker logs -f 를 사용해여 컨테이너 로그출력 & follow

```bash
$ docker logs -f {{container_id}}
```

### docker ps를 사용하여 상태출력

```bash
$ docker ps
$ docker ps -a
```

### docker stop을 사용하여 container exit

```bash
$ docker stop {{container_id}}
```

* or `rm`

### docker images를 사용하여 이미지 목록보기

```bash
$ docker images
```









## Dockerfile

> docker image를 만들기 위한 description

### File

```dockerfile
FROM zironycho/pytorch:1.1.0-slim

WORKDIR /workspace

COPY requirements.txt .
RUN pip install -r requirements.txt && rm -rf /root/.cache/pip
RUN apt get update
```

* cache: file status



![](https://t1.daumcdn.net/cfile/tistory/2567453B5214F0D815)



### build image

```bash
$ docker build -t my-pytorch:v1 -f docker/Dockerfile .
```



### Tips

* ignore file/dir when it build 

   * `.dockerignore` 를 이용하여 image를 생성할 때 layer에 포함시키지 않기

 ```ini
 **/__pycache__
 **/.idea
 **/generated
 .pytest_cache
 **/*.pyc
 **/*.wav
 **/*.png
 **/*.t7 
 ```

* 필요한 파일만 copy하기.

   * union file system
   * example) copy again, again, again 









## docker-compose

> config file for multi-container applications

### File

```yaml
version: "3.7"

services:
  train-1:
		build:
			context: .
			dockerfile: ./docker/Dockerfile
		command: python train.py
    volumes:
    - .:/workspace
    - my-database:/database
    environment:
    - MAX_EPOCH=100
    - NVIDIA_VISIBLE_DEVICES=0

	train-2:
    image: zironycho/pytorch-jupyter:1.1.0-slim
    environment:
    - HELLO=WORLD
    - NVIDIA_VISIBLE_DEVICES=1,2
    ports:
    - "8888:8888"



volumes:
  my-database:
  	driver: local

```

![](https://t1.daumcdn.net/cfile/tistory/243B6E4A5215B89430)







## docker and nvidia-docker

> nvidia gpu를 사용하기 위해서는 nvidia-docker를 사용함.

```bash
$ docker run --runtime=nvidia \
	--rm \
	nvidia/cuda:9.0-base nvidia-smi
```



## Refs:

* https://docs.docker.com/engine/docker-overview/
* https://docs.docker.com/engine/reference/builder/
* https://docs.docker.com/compose/overview/
* [http://haanjack.github.io/docker/2018/04/14/nvidia-gpu-isolation.html](http://haanjack.github.io/docker/2018/04/14/nvidia-gpu-isolation.html)
* https://judekim.tistory.com/15

