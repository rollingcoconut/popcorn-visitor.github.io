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
sudo service ssh status
# sudo service ssh start
```

- Turned on the network inteface and assigned IPs if they were missing. 
_We determined if a popcorn instance had no IP through _ifconfig_._

It's worth noting that in VirtualBox the default gateway is assigned 10.0.2.2, and first IP assigned is 10.0.2.15, which was given to our host VM. 

__X86 popcorn__
```
sudo ifconfig enp0s3 10.0.2.16 netmask 255.255.255.0                  
sudo route add default gw 10.0.2.2 enp0s3 
```

__ARM popcorn__
```
sudo ifconfig enp0s1 10.0.2.17 netmask 255.255.255.0                  
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

(_image taken from popcorn linux tutorial_)
![image](https://user-images.githubusercontent.com/17166431/128949977-a8273c25-0994-4d63-8bb2-fd70006949ca.png)



and this image of Process migration between kernels.

(_image taken from popcorn linux tutorial_)
![image](https://user-images.githubusercontent.com/17166431/128950038-a78df103-914e-4fee-8862-64e1f8a5a6d3.png)


Our popcorn-hello program output is below 

![image](https://user-images.githubusercontent.com/17166431/128950134-2c31c99c-bebd-4bdf-8741-c3dfb5df0852.png)

as well as the dmesg kernel buffer log.
![image](https://user-images.githubusercontent.com/17166431/128950153-c938a24e-b397-4e5f-ac46-ff28c5285c9e.png)


### Experimentation

A distributed kernel Operting System can offer interesting benefits. Robert Lyerly writes in his dissertation for Popcorn Linux that CPUs with different ISAs offer varied benefits (power efficiency, performance); the capacity to migrate applications across heterogeneous-ISA CPUs can allow for "optimizing a variety of workloads."

"Application migration allows the system software to optimize how a given workload executes in the system to best utilize the available compute resources, e.g., placing applications in consideration of architectural characteristics or multiprogrammed environments."[[1]](https://vtechworks.lib.vt.edu/bitstream/handle/10919/100599/Lyerly_RF_D_2019.pdf?sequence=1&isAllowed=y)


Instead of migrating (ie kernel hopping) applications, Popcorn Linux also allows developers to migrate pthreads. Migrating user process threads across kernels is highly fascinating as it is not yet a usual load balancing paradigm. 

With that in mind, we wanted to test a very simple hypothysis: 
- Running N user process pthreads over several popcorn-kernels should be more effective than running N user process pthreads on only one popcorn kernel for compute intensive jobs.

![Screenshot from 2021-08-11 00-08-50](https://user-images.githubusercontent.com/17166431/128968380-79bd62ab-604a-4be2-bc43-f7f658982c10.png)

To test our hypothysis we created a number counting job where the number of threads and load per thread were configurable. As we increased the load we expected to see the jobs with more threads behave about the same when running on only one virtual machine -- threads don't help the performance when you only have one core.

When we allowed the number-counting job to "migrate threads" and balance them across the two VMs, we expected to see improved performance that we understood would be offset by the cost of message passing between the two kernels.  


#### Observations

We did not observe the expected behaviour. While we don't deny the fact that it could just have been our setup not being powerful enough, we made some some observations during testing.

1. The x86 machine consistently took longer to run the compiled number crunching code
2. The reccommended migration procedure corrupted the C program when dealing with larger compute loads 

(_image taken from popcorn linux tutorial_)

![Screenshot from 2021-08-11 14-52-55](https://user-images.githubusercontent.com/17166431/129086448-a6eb0aed-a829-4fc7-aee4-d5e84da0374f.png)

Initially we migrated threads manually, issuing a 35 signal to half of the created pthreads on the ARM-popcorn VM. When the compute load was large, the popcorn-compiled program on the ARM VM often went into a defunct state.

![image](https://user-images.githubusercontent.com/17166431/129086959-9dc2e9fd-f5b6-4c78-9631-d10b90104c04.png)

This behaviour persisted even when using the thread migration scheduler provided in the tutorial.

When migration is enabled, the program crashes on the ARM machine when the compute load exceeded 80K weighted operations. 


![Screenshot from 2021-08-11 16-37-57](https://user-images.githubusercontent.com/17166431/129099782-ab04fbe0-de14-4571-bb84-b1e015bcf850.png)

![Screenshot from 2021-08-11 14-41-03](https://user-images.githubusercontent.com/17166431/129085435-7f65c782-debf-43a7-842a-dec2675654c0.png)

![Screenshot from 2021-08-11 14-42-49](https://user-images.githubusercontent.com/17166431/129085447-a9b6bb04-ef4f-4154-930c-18934c6b2708.png)

While we cannot precisely explain why the x86 VM ran more slowly than the ARM machine, we think it is safe to assume that having the migration scheduler run on the x68 VM could have contributed latency.

We also suspect that just killing a thread to force pthread migration was not sufficient to keep program stability -- something was happening during the popcorn thread kill and/or popcorn thread migration that brought down the whole program.


### Conclusion

The tutorial followed and modified for experimentation was produced in 2019. It is possible that work has been done on this developing Operating System since then to enable safer pthread migration, or alternate reccommendations made on how to migrate pthreads.

We none the less had a blast playing with the Popcorn Linux Operating System.  


### References
[1](https://vtechworks.lib.vt.edu/bitstream/handle/10919/100599/Lyerly_RF_D_2019.pdf?sequence=1&isAllowed=y) R. F. Lyerly, “Popcorn linux: A compiler and runtime for state transformation between heterogeneous-ISA architectures,” Ph.D. dissertation, 2016. [Online]. Available: https://vtechworks.lib.vt.edu/bitstream/handle/10919/100599/Lyerly_RF_D_2019.pdf

