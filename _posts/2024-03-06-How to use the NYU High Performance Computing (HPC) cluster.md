---
layout: post
category: how-to
---

All NYU folks have access to the Greene High Performance Computing (HPC) cluster. It's super powerful (\> 30K Intel CPU cores and > 360 NVIDIA GPUs and AMD GPUs—see specs [here](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/hardware-specs?authuser=0)) and can handle with grace many things that your humble personal machines will take forever to do, if they can do it at all. 

It's also surprisingly *easy* to use, contrary to what a cursory glance at the official [website](https://sites.google.com/nyu.edu/nyu-hpc/home?authuser=0) may suggest. To be clear, the website is full of useful information, but perhaps too much of it. A simple-minded user like me would probably appreciate a one-pager with the minimum amount of information needed to *make it work*. Hence this how-to guide. 

*For all the steps below, use [NYU VPN](https://www.nyu.edu/life/information-technology/infrastructure/network-services/vpn.html) if you're off campus.* 

### Step 0: Make an account

Follow the instructions [here](https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc/getting-and-renewing-an-account/requesting-an-hpc-account-with-iiq?authuser=0) to request an account. 

### Step 1: Upload your files

The HPC has its own data storage systems. You need to upload your data and/or scripts there to access them while using the HPC (read: your local files are not accessible to the cluster).

* You'll probably use ```/home```, ```/scratch```, and ```/archive``` the most. There're other spaces that you can read more about [here](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/hpc-storage/data-management?authuser=0).

|      | Space    |      | Space Purpose                                         | Backed Up / Flushed (routinely removed)     | Quota Disk Space / # of Files |
| :--: | -------- | ---- | ----------------------------------------------------- | ------------------------------------------- | ----------------------------- |
|      | /home    |      | Personal user home space that is best for small files | YES / NO                                    | 50 GB / 30 K                  |
|      | /scratch |      | Best for large files                                  | **NO /** **Files not accessed for 60 days** | 5 TB / 1 M                    |
|      | /archive |      | Long-term storage                                     | YES / NO                                    | 2 TB / 20 K                   |



* You probably want to upload the files for your current project to ```/scratch```. Note that there're no back ups, and if you don't access a file in 60 days, it'll be removed.

There're [many ways](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/hpc-storage/data-management/data-transfers?authuser=0) to upload your files. I recommend the following:

* For *large* files (> 50 mb): Follow the instructions for using [Globus](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/hpc-storage/data-management/data-transfers/globus?authuser=0).
* For *small* files (<= 50 mb): Use the graphical interface, Open OnDemand
  * Go to https://ood.hpc.nyu.edu/ and log in using your NYU credentials.
    * *Make sure to use NYU VPN if off campus*
  * On the top banner, select ```Files```. Select your folder.

<img src="assets/img/Screenshot 2024-03-06 at 13.00.33.png">

​		Use the interface to upload and download files.

<img src="assets/img/Screenshot 2024-03-06 at 13.03.08.png">

### Step 2: Access HPC

There are [many ways](https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc?authuser=0#h.7kawz2pfzl9d) to access HPC. We'll use the graphical interface, Open OnDemand, here because it's the easiest.

Go to https://ood.hpc.nyu.edu/ and log in using your NYU credentials.

* *Make sure to use NYU VPN if off campus* 

On the top banner, select ```Interactive Apps``` to see the different web-based applications you can use. 

* We will use RStudio Server as an example.

<img src="assets/img/Screenshot 2024-03-06 at 11.53.36.png">

### Step 3: Start a session

Change the following settings to suit your needs and click `Launch`. If you're confused about how many cores/how much memory/which GPU type (if any) to request, experiment!

* I would recommend *requesting more hours* than you think you'll need. This is because there is no way to extend a running session. So suppose your very-complex-model needs maybe 30 more minutes to converge but your session ends in 5 minutes, too bad!
  * That said, don't request too many hours. This is because your session will stay longer in the queue before it can start.
* The HPC monitors your usage to make sure that you're not wasting resources. For example, if you requested GPU but have not been using it for a while, the HPC will kick you out.
* See [FAQs](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/best-practices?authuser=0).

<img src="assets/img/Screenshot 2024-03-06 at 13.28.37.png">


When your session is ready to start, you can find it in ```My Interactive Session``` on the top banner. 	

<img src="assets/img/Screenshot 2024-03-06 at 13.09.40.png">

You made it! 🎉

> One word on parallelizing your code:
>
> * Just because you requested multiple cores doesn't mean that you're using them. You need to write your code so that it uses all available cores. This is called *parallelization*.  
> * In Python, you can use the[ ```multiprocessing```](https://docs.python.org/3/library/multiprocessing.html) package. In R, you can use the [```furrr```](https://furrr.futureverse.org) package.
> * There're many detailed tutorials for parallelizing in [Python](https://www.sitepoint.com/python-multiprocessing-parallel-programming/) and [R](https://bookdown.org/rdpeng/rprogdatascience/parallel-computation.html). 