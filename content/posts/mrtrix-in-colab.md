+++
title =  "Unleashing the dragon: running MRtrix3 in Google Colab"
tags = ["dMRI", "MRtrix3", "tips-and-tricks"]
date = "2021-02-03"
image = "img/drive_peek.gif"
caption = "Just a normal day in [2034](https://www.commitstrip.com/en/2016/12/22/terminal-forever/), browsing through the MRI data in my Google Drive and peeking at it."
+++


## A day in the life of browsers


A random thought crossed my mind last week: with Javascript at the [top of programming languages rankings](https://redmonk.com/sogrady/2020/07/27/language-rankings-6-20/ 'Javascript did it again') and with so many [notebook systems](https://pg.ucsd.edu/publications/computational-notebooks-design-space_VLHCC-2020.pdf 'Literally dozens of notebook systems') covering most needs, how far are we from doing _almost_ everything in a browser? Do we actually _want_ to do everything in a browser? Postponing the _dystopian_ consequences to the next _Black Mirror_ season, it is true that browsers have become so _universal_ that it makes sense to give access to resources through them. [Google Colab](https://colab.research.google.com 'Colab') is a good example: one has access for free to virtual machines with GPUs or even TPUs through a notebook system. Getting closer to brains, it may seem a limitation that you need to access to all of that mainly through Python - what about other software packages? What if we need to preprocess data before feeding them to neural networks for training purposes? In this post, I will go through a pratical example with [MRtrix3](https://www.mrtrix.org 'MRtrix'), showing how you can install it, how not to have to re-install it every time, and that there may be _more_ to Colab than just notebooks.

The Jupyter Notebook to reproduce this procedure is on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/tips-and-tricks/mrtrix3-in-colab), together with some advices in case you want to try to variate a bit and install ANTs or FSL. You can try all this out directly using this [Google Colab link](https://colab.research.google.com/github/matteomancini/neurosnippets/blob/master/tips-and-tricks/mrtrix3-in-colab/mrtrix3_in_colab.ipynb 'Open in Colab').


## The Colab-Drive alliance


The steps to have MRtrix3 up and running within Colab are pretty much the ones described in the [official documentation](https://mrtrix.readthedocs.io/en/latest/installation/build_from_source.html). But there's a catch: we do not want to reinstall MRtrix3 everytime our Colab session is over. To avoid this, we will use a nice trick - mounting the Google Drive volume directly on Colab and storing MRtrix3 there:

```
from google.colab import drive
drive.mount('/gdrive')

%cd /gdrive/MyDrive

!apt-get install libeigen3-dev libqt5opengl5-dev libqt5svg5-dev libfftw3-dev
!git clone https://github.com/MRtrix3/mrtrix3.git

%cd mrtrix3
!chmod +x configure
!chmod +x build
!./configure -nogui

!./build
```

This will take a while. As you can see, apart from getting access to our Drive, the steps are the usual ones for MRtrix3 _aficionados_. The only differences here are that we are giving up the GUI, and the list of packages installed with `apt-get` is slightly shorter since some of the libraries are already there.
Once these steps are done, there is no need to repeat them. Actually, if you do not need to compile a different version of MRtrix3, there is no need to do them at all, since you can directly download the files from [this link](https://drive.google.com/file/d/1AppHBa9cPz2dQsIX8-Ca8r15KGasluFn/view?usp=sharing 'MRtrix3 for everyone!') and put them into your own Drive.
So once the files are uploaded, here is the first step to do every time you want to use MRtrix3 in Colab:

```
import os
os.environ['PATH'] += ":/gdrive/MyDrive/mrtrix3/bin"
!chmod +x /gdrive/MyDrive/mrtrix3/bin/*
```

Now that the system path includes the MRtrix3 binaries and we made sure that they are executable, let's test if things are working. I used [`dipy`](https://dipy.org 'DiPy') to download some data, and then estimated the orientation distribution function (ODF) and the fiber orientation distribution (FOD):

```
%cd /root
!pip install dipy

from dipy.data import fetch_sherbrooke_3shell

fetch_sherbrooke_3shell()

# Time for some processing!
!dwi2response dhollander -fslgrad /root/.dipy/sherbrooke_3shell/HARDI193.bvec /root/.dipy/sherbrooke_3shell/HARDI193.bval /root/.dipy/sherbrooke_3shell/HARDI193.nii.gz wm.txt gm.txt csf.txt

!dwi2fod msmt_csd -fslgrad /root/.dipy/sherbrooke_3shell/HARDI193.bvec /root/.dipy/sherbrooke_3shell/HARDI193.bval /root/.dipy/sherbrooke_3shell/HARDI193.nii.gz wm.txt wm.nii.gz gm.txt gm.nii.gz csf.txt csf.nii.gz
```

Done! Unfortunately we are not able to look at the FOD using for example the amazing [FURY](https://fury.gl 'The perfect companion to dipy') library, as for some reason VTK-based tools do not work in Colab. We could use `matplotlib` to have a look at things like simple slices, but there's something cooler than that: [mrpeek](https://github.com/MRtrix3/mrpeek 'Peeking at MR data') - peeking at the data directly in the terminal (or at least in one that supports [sixel encoding](https://github.com/MRtrix3/mrpeek/wiki 'Check the terminal requirements')). We can easily build it in Colab:

```
%cd /gdrive/MyDrive
!git clone https://github.com/MRtrix3/mrpeek.git
%cd mrpeek
!../mrtrix3/build
```

But then we would need access from a terminal, and we don't have that here. Or do we?


## Going deep down the rabbit hole

I started this post talking about doing _almost everything_ in a browser, but actually it is not always what we want. Take this very specific case for instance: to run commands in the terminal we need to remember every time to put an exclamation mark at the beginning. Why can't we just use a terminal? Well, yes - we can! It is a matter of:

1. setting an SSH server in our Colab instance;
2. exposing it in some way.

The first one is quite feasible, after all we have root privileges within the instance; and we can pull the second one off using a technique called [_reverse tunneling_](https://www.howtogeek.com/428413/what-is-reverse-ssh-tunneling-and-how-to-use-it/ A brief explanation of reverse tunneling), where we basically connect from Colab to a strategic server, and then we use that connection to get to Colab itself. To do this we will use [ngrok](https://ngrok.io/), which is free for our simple needs. So let's get started:

```
# Generate root password
import random, string
password = ''.join(random.choice(string.ascii_letters + string.digits) for i in range(20))

# Download ngrok
!wget -q -c -nc https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
!unzip -qq -n ngrok-stable-linux-amd64.zip
# Setup sshd
!apt-get install -qq -o=Dpkg::Use-Pty=0 openssh-server pwgen > /dev/null
# Set root password
!echo root:$password | chpasswd
!mkdir -p /var/run/sshd
!echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
!echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
!echo "LD_LIBRARY_PATH=/usr/lib64-nvidia" >> /root/.bashrc
!echo "export LD_LIBRARY_PATH" >> /root/.bashrc

# Run sshd
get_ipython().system_raw('/usr/sbin/sshd -D &')

# Ask token
print("Copy authtoken from https://dashboard.ngrok.com/auth")
import getpass
authtoken = getpass.getpass()
```

We have just taken care of both setting up `sshd` and installing `ngrok`. Since we have also got the authentication token, we just need to launch `ngrok`, and retrieve the address and password we will use to connect:

```
# Create tunnel
!chmod +x ngrok
get_ipython().system_raw('./ngrok authtoken $authtoken && ./ngrok tcp 22 &')
# Get public address
!curl -s http://localhost:4040/api/tunnels | python3 -c \
    "import sys, json; print('SSH to: root@', json.load(sys.stdin)['tunnels'][0]['public_url'][6:], sep='')"
# Print root password
print("Root password: {}".format(password))
```

That's it! We can now connect to our Colab instance through SSH and use `mrpeek` from there if we want to. But probably at this point it is clear that a lot can be done in this way (just to throw there the first two ideas that come to mind: processing that bunch of data you had piling up on your Drive? Or, I don't know, using a _TPU_ runtime?).


## Useful references

* [ANTs in Colab](https://www.suyogjadhav.com/misc/2019/03/28/Using-ANTs-package-on-Google-Colaboratory/)

* [mrpeek - for a life in the terminal ](https://github.com/MRtrix3/mrpeek)

* [Usage example of ngrok to expose local resources](https://netdevops.me/2020/easily-exposing-your-local-resources-with-ngrok-and-fwd/)

* [More Google Colab tips and tricks](https://github.com/shawwn/colab-tricks)

* [For those who want to run Jupyter Lab in Colab (!)](https://imadelhanafi.com/posts/google_colal_server/)
