+++
title =  "One container to rule them all: FSL-FIX in a box"
tags = ["fsl", "docker", "containers", "fMRI"]
date = "2021-01-27"
image = "img/vm_vs_docker.svg"
caption = "A wedding-cake plot of virtual machines against containers. Still not convinced? How about [software enlightenment](https://xkcd.com/1988/)?"
+++


## "Use a container" is the new "You need a virtual machine"

I have to be sincere: the first time I heard about containers I was not impressed. It sounded like an _exotic_ approach, and I couldn't see why would you need that. A few years later, I have changed my mind a lot about it: not only containers can be used to create reproducible workflows, but they are also useful in several applications ([taken to the extreme in some cases](https://jonathan.bergknoff.com/journal/run-more-stuff-in-docker/ 'Running almost everything in docker (!)')). In this post, I will explore the idea of using a container in a scenario where usually one would have relied on a virtual machine. If after reading this post you still think you need a virtual machine, have a look first at the amazing [NeuroDesk](https://github.com/NeuroDesk/neurodesk 'NeuroDesk repository').

The idea for this post came to me when a friend of mine reached out asking for help with a tool that I did not use in a long time: [`FIX`](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FIX 'FIX page on the FSL website'). For those of you who are not familiar with it, it's a collateral tool to [`FSL`](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/ 'FSL homepage') that can be used to automatically classify the independent components estimated from fMRI data using `melodic`. To tailor this classification problem, the tool itself can be used for training the classifier on a dataset of interest. My friend, who has just started in the neuroimaging world, was struggling to get it running in a virtual machine. After a bit of _remote_ testing and several error messages, I had an idea: why not just using a container? Here I will show the `Dockerfile` I came up with and how it can be used not only to run `FIX`, but also any `FSL` tool. Both the `Dockerfile` and the related script are on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/workflows/fslfix-in-a-box). 



## A few hours of trial-and-error later...

I will not lie: making a `Dockerimage` for an _articulated_ piece of software is not easy. And `FIX` is for sure _articulated_: it combines `bash`, `R` and `matlab`/`octave`. It is an interesting implementation of the proverb ["With great power there must also come great responsibility"](https://en.wikipedia.org/wiki/With_great_power_comes_great_responsibility 'Wikipedia') in software. As a result of these _multimodal_ dependencies, making a container out of it can take time. In my case, I think it was a few hours of trial-and-error, but once it is done, you _cannot break it anymore_ (unless you have the _unfortunate_ idea of touching it again). So let's have a look at the sections of the `Dockerfile`.

The starting point is picking a base image. Let's start from a _user-friendly_ GNU/Linux distro, and with a stable, supported version -- we will go with Ubuntu Bionic Beaver. This first block in the Dockerfile will take care of pulling this image and install the fundamental stuff (e.g. `zip`/`unzip` to handle archives, `wget` to download files, etc.):

```

FROM ubuntu:bionic-20201119

# Install fundamental packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    lsb-core \
    bsdtar \
    zip \
    unzip \
    gzip \
    curl \
    jq \
    wget \
    python-pip \
    software-properties-common && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
    
```

Next on the list is `R`. An important thing to do before installing `r-core` and the related packages is to set up the _locale_ settings (mostly timezone and character encoding), otherwise we would be prompted to insert those from the keyboard and we would be stuck. Another important thing is to make sure to get the right version of each package, and to handle the dependencies. Luckily, the `FIX` documentation is [quite detailed about it](https://git.fmrib.ox.ac.uk/fsl/fix/-/blob/master/README). Here it goes:


```

#############################################
# Download and install R and necessary packages

ENV TZ=Europe/Rome
ENV LC_ALL C.UTF-8
ENV LANG C.UTF-8
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone && \
    apt-get update && apt-get install -y --no-install-recommends r-base r-cran-devtools \
        libblas-dev liblapack-dev gfortran r-cran-catools r-cran-gplots g++ && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    Rscript -e 'require(devtools); install_version("kernlab", version="0.9-24")' && \
    Rscript -e 'require(devtools); install_version("ROCR", version="1.0-7")' && \
    Rscript -e 'require(devtools); install_version("class", version="7.3-14")' && \
    Rscript -e 'require(devtools); install_version("mvtnorm", version="1.0-8")' && \
    Rscript -e 'require(devtools); install_version("multcomp", version="1.4-8")' && \
    Rscript -e 'require(devtools); install_version("coin", version="1.2-2")' && \
    Rscript -e 'require(devtools); install_version("party", version="1.0-25")' && \
    Rscript -e 'require(devtools); install_version("e1071", version="1.6-7")' && \
    Rscript -e 'require(devtools); install_version("randomForest", version="4.6-12")'

```

It is time for FIX! It is mainly a matter of downloading it and _unpacking_ it where we want it to be (I opted for `/opt` for no specific reason):


```

#############################################
# Download and install FIX

RUN wget -v http://www.fmrib.ox.ac.uk/~steve/ftp/fix.tar.gz && \
    mkdir -p /tmp/fix && \
    cd /tmp/fix && \
    tar zxvf /fix.tar.gz --exclude="compiled/Darwin/" && \
    mv /tmp/fix/fix* /opt/fix && \
    rm /fix.tar.gz && \
    cd / && \
    rm -rf /tmp/* /var/tmp/*

ENV FSL_FIXDIR=/opt/fix
ENV PATH "$PATH:/opt/fix"

```

Next on the list is Matlab Compiler Runtime. As for the `R` packages, one needs to pay attention to the specific version required and download _that_ one, and also once again avoid _interactive_ steps:


```

#############################################
# Download and install Matlab Compiler Runtime

RUN wget https://ssd.mathworks.com/supportfiles/downloads/R2017b/deployment_files/R2017b/installers/glnxa64/MCR_R2017b_glnxa64_installer.zip && \
    mkdir /opt/mcr && mkdir /mcr-install && cd mcr-install && \
    unzip /MCR_R2017b_glnxa64_installer.zip && \
    rm -rf /MCR_R2017b_glnxa64_installer.zip && \
    ./install -destinationFolder /opt/mcr -agreeToLicense yes -mode silent && \
    cd / && rm -rf mcr-install

ENV FSL_FIX_MCRROOT=/opt/mcr

```

We are at the last chunk! Here we will be installing `FSL` from the NeuroDebian repository, in particular `fsl-core` and `fsl-atlases`, but also `OpenJDK` (otherwise `mcr` will complain). After setting all the `FSL`-related variables, we will add a non-root user (I mean, we don't need to be root to process fMRI data, so let's just use a normal user as we would do in a _real_ machine):

```

#############################################
# Download and install FSL 5 + OpenJDK

RUN apt-get update && apt-get install -y dirmngr && \
    wget -O- http://neuro.debian.net/lists/bionic.de-fzj.full | \
    tee /etc/apt/sources.list.d/neurodebian.sources.list && \
    apt-key adv --recv-keys --keyserver hkp://pool.sks-keyservers.net:80 0xA5D32F012649A5A9 && \
    apt-get update && apt-get install -y fsl-core fsl-atlases openjdk-8-jdk && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Configure FSL environment
ENV FSLDIR=/usr/share/fsl/5.0
ENV FSL_DIR="${FSLDIR}"
ENV FSLOUTPUTTYPE=NIFTI_GZ
ENV PATH=/usr/lib/fsl/5.0:$PATH
ENV FSLMULTIFILEQUIT=TRUE
ENV POSSUMDIR=/usr/share/fsl/5.0
ENV LD_LIBRARY_PATH=/usr/lib/fsl/5.0:$LD_LIBRARY_PATH
ENV FSLTCLSH=/usr/bin/tclsh
ENV FSLWISH=/usr/bin/wish
ENV FSLOUTPUTTYPE=NIFTI_GZ

#############################################
# Add non-root user

RUN useradd --create-home --shell /bin/bash fsluser
USER fsluser
WORKDIR /home/fsluser
ENV USER=fsluser

```

And the `Dockerfile` is complete! Now it is a matter of building and tagging it. As I was planning to upload it to DockerHub, I included also my username in the tag:

```

docker build -t ingmatman/fslfix:latest .

```

This will take a while, and if you pay attention to the output you can glimpse signs of each step we wrote. Since I have already built and uploaded this image, if you do not want to play with it or do not need to change anything, you can instead pull the whole thing from [DockerHub](https://hub.docker.com/repository/docker/ingmatman/fslfix):

```

docker pull ingmatman/fslfix:latest

```

And the size is... 9.43GB! That's big! But once you realise that you have the _whole_ `FSL` (including the atlases), `FIX`, `R` and `mcr` in there, it does not look that big anymore. After all, we are doing this to avoid using a virtual machine, and a virtual machine probably would have taken that much space.


## Welcome to the Machine

So now we can use a container to run `FIX`. To test it, we need some data. As sample data, I have chosen the [Midnight Scan Club dataset](https://openneuro.org/datasets/ds000224/versions/1.0.3 'MSC dataset') (you gotta love the almost _speakeasy_ name) shared by Evan Gordon and colleagues on [OpenNeuro](https://openneuro.org 'OpenNeuro'). As usual, this dataset can be easily downloaded through [DataLad](https://www.datalad.org 'DataLad'):

```
datalad install https://github.com/OpenNeuroDatasets/ds000224.git

datalad get -J 4 ds000224/sub-MSC01/ses-*01/

```

As we will be using as a test just the resting-state data (and the related T1-weighted volume) from the first subject, let's copy those files in a single folder to make things easier in a bit:

```

mkdir fixdata

cp ds000224/sub-MSC01/ses-func01/func/sub-MSC01_ses-func01_task-rest_bold.nii.gz fixdata/sub01_func_rs.nii.gz

cp ds000224/sub-MSC01/ses-struct01/anat/sub-MSC01_ses-struct01_run-01_T1w.nii.gz fixdata/sub01_anat_t1w.gz

```

So now we can preprocess the data. One way to do it is through the GUI in `Melodic`:

```
docker run --net="host" --env="DISPLAY" -v="$HOME/.Xauthority:/home/fsluser/.Xauthority:rw" \
    -v="$HOME/fixdata:/home/fsluser/data:rw" ingmatman/fslfix bash -c "Melodic"
```

We have just done two important things:

1. we mounted the file `.Xauthority` from the host in the container -- this will actually make possible to see the GUI running in the container;

2. we mounted the data folder from the host in the container.

Although the `.Xauthority` trick works in Linux, in macOS and in Windows you actually need to install an X server -- more details are available in [this post](https://cuneyt.aliustaoglu.biz/en/running-gui-applications-in-docker-on-windows-linux-mac-hosts/ 'Running GUI applications in Docker on Windows, Linux and Mac hosts').
Once we have completed the configuration for the preprocessing, we can either save the configuration in `fsf` format or run it directly. I would suggest to save it and then run it using `melodic` (as done here), since the GUI will remain open (and therefore the container will keep running) even after the preprocessing has finished, while running just `melodic design.fsf` terminates once the preprocessing is done.
As I have already prepared the configuration file, we can just copy it where we need it and then proceed:

```
cp design.fsf fixdata/

docker run --net="host" --env="DISPLAY" -v="$HOME/.Xauthority:/home/fsluser/.Xauthority:rw" \
    -v="$HOME/fixdata:/home/fsluser/data:rw" ingmatman/fslfix bash -c "melodic data/design.fsf"
```

Since the output folder is shared between the container and the host, we can keep track of what is happening in a browser through the `report_log.html` file. Once the preprocessing is done, we are ready to use `FIX`:

```
docker run --net="host" --env="DISPLAY" -v="$HOME/.Xauthority:/home/fsluser/.Xauthority:rw" \
    -v="$HOME/fixdata:/home/fsluser/data:rw" ingmatman/fslfix bash -c "fix data/results.ica /opt/fix/training_files/Standard.RData 20"
```

That's it!
One final note: a good habit is to set up an entry point for the container, so one doesn't have to worry about additional stuff to remember. However, in this case I omitted that on purpose: here you have the whole `FSL` in there, and you can run any of its tools, as you would do in a virtual machine.


## Useful references


* [A good intro post on Docker](https://medium.com/analytics-vidhya/starting-with-docker-bfd74021d5c7 'From Jennifer Stark')

* [FIX documentation](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FIX 'FIX page on the FSL website')

* [Midnight Scan Club dataset](https://openneuro.org/datasets/ds000224/versions/1.0.3 'OpenNeuro')

* [Next-level containers to replaces virtual machines: NeuroDesk](https://github.com/NeuroDesk/neurodesk 'NeuroDesk repository')

