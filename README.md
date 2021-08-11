## Exploring the Popcorn Linux Operating system
### A distributed kernel OS supporting heterogeneous architecture

[PopcornLinux](http://www.popcornlinux.org/index.php/overview) is a developing Operating System that can run transparently over a heterogeneous set of kernels (x68/arm)  on heterogeneous architecture. The different kernels communicate entirely through message passing, and allow entire applications as well as user process threads to be "migrated" from one kernel to another, even when the kernels are located on machines of different ISAs. 

We wanted to take this developing Operating System for a test-drive and get a feel for message latency, popcorn development struggles, and the very cool experience of using **distributed kernels** first hand. Here we document the the additional steps we needed to follow along with the tutorial, as well as the experiment we performed once we got things running. 

[Popcorn Linux tutorial](https://github.com/ssrg-vt/popcorn-compiler/tree/main/tutorial/sosp-2019)


### Starting up Popcorn Linux 
After setting up a local VM and downloading the x86.img and arm.img file (or just using the .ova file provided), we applied the following VirtualBox configurations:

- Start VirtualBox in bridged mode with virtualization turned on. 
```
[from cmd.exe] $VBoxManage modifyvm popcorn-box --nested-hw-virt on
```
![image](https://user-images.githubusercontent.com/17166431/128944034-f5f81ebf-c22f-4d96-a7e7-a9a80d4c5ddd.png)
![image](https://user-images.githubusercontent.com/17166431/128944093-dc45dd68-df86-46c9-97a4-e0ebf28fd5b7.png)


As instructed in the tutorial we use the qemu emulator to start running two instances of our Popcorn Linux OS, an ARM-compiled instance as well as an x86.


![image](https://user-images.githubusercontent.com/17166431/128944820-bc4bc650-b5a4-4f85-afdf-77b4cccae898.png)
![image](https://user-images.githubusercontent.com/17166431/128946061-210a0170-9da4-46c7-b444-bbda9fc21e8d.png)


Since the two instances were required to communicate with each other, we also had to do the following, which was not documented in the tutorial.

- Confirmed ssh service was running on both machines
```
sudo servic ssh status
# sudo service ssh start
```

- Turned on the network inteface and assigned IPs if they were missing. 
_We determined if a popcorn instance had no IP through _ifconfig_._

It's worth noting that in VirtualBox the default gateway is assigned 10.0.2.2, and first IP assigned is 10.0.2.15, which was given to our host VM. 

__X86 popcorn__
```
sudo ifconfig enps0s3 10.0.2.16 netmask 255.255.255.0                  
sudo route add default gw 10.0.2.2 enp0s3 
```

__ARM popcorn__
```
sudo ifconfig enps0s1 10.0.2.17 netmask 255.255.255.0                  
sudo route add default gw 10.0.2.2 enp0s1
```


__Our VM setup__
![image](https://user-images.githubusercontent.com/17166431/128946584-4486fd35-b19b-46fa-b26f-60c2dd8a8dec.png)


__Enable Message Passing__

Next, as instructed in the tutorial, we used _insmod_ to add the popcorn OS messaging kernel to both machine instances.
```
cd /root/[linux-x86 | linx-arm]/msg_layer/
sudo insmod ./msg_socket.ko ip=10.0.2.16,10.0.2.17
```
After installing the messaging module into the kernel, each instance can identify its peers.

![image](https://user-images.githubusercontent.com/17166431/128947291-cc4617f8-f0fa-4eb9-bb16-38f858262fff.png)


At this point we had two working instances of Popcorn Linux (one running ARM ISA, the other x86), and were able to perform a popcorn hello. 

__Hello Popcorn__

Using the provided __popcornhelloS.c__ program,  the popcorn compiler creates binaries for both architectures then scp's the arm binary to the arm-popcorn instance. The _entire_ "hello world" process (including user and kernel state threads) is then continually programmed to run back and forth between each kernel. 

The tutorial includes this rather helpful image of the merged binaries

(_taken from popcorn linux tutorial_)
![image](https://user-images.githubusercontent.com/17166431/128949977-a8273c25-0994-4d63-8bb2-fd70006949ca.png)



and this image of Process migration between kernels.

(_taken from popcorn linux tutorial_)
![image](https://user-images.githubusercontent.com/17166431/128950038-a78df103-914e-4fee-8862-64e1f8a5a6d3.png)


Our popcorn-hello program output is below 

![image](https://user-images.githubusercontent.com/17166431/128950134-2c31c99c-bebd-4bdf-8741-c3dfb5df0852.png)

as well as the dmesg kernel buffer log.
![image](https://user-images.githubusercontent.com/17166431/128950153-c938a24e-b397-4e5f-ac46-ff28c5285c9e.png)


### Experimentation

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/rollingcoconut/popcorn-visitor.github.io/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
