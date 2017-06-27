
## UPDATE

I recently found the following discussion on one of the Intel discussion forums:

[Problems with reduction done in CPU](https://software.intel.com/en-us/forums/opencl/topic/558984)

The key bit is as follows:

> The problem is in the kernel code, at line 18:
>
>       data[get_group_id(0)] = partial_sums[0];
>
> Basically, this logic expects that work-groups are executed sequentially, but this is not the case (for CPU at least). What happens is that some work-group with a higher index might be executed earlier and so it rewrites the data that has not been processed yet by the lower index (say 0) work-group.
>
> One obvious solution would be to write temporary reduction sums to indexes above the current global size (of course allocating necessary amount of space first) and adjust offsets for each iteration of reduction.

*A working version of the code in C++ can be found [here](https://github.com/taylorjg/ReductionCpp).*

## Description

Recently, I have been following along with the code in Chapter 10 of _OpenCL in Action_ (sections 10.2 and 10.3).
I am using C# and [OpenCL.NET](https://openclnet.codeplex.com/). Everything was fine until I got to the
bit that uses a second kernel to do the final additions on the device rather than on the host. The code
works fine on my GPU but gives a variety of wrong answers on the CPU.

In order to investigate further, I downloaded the
[source code](https://manning-content.s3.amazonaws.com/download/8/56a2ab3-4fe2-440b-8db1-bd5fa93deec6/source_code_vs2010.zip)
that accompanies the book.
I made two tiny tweaks to the `Ch10_reduction_complete` example in order to force the
code to run on the CPU rather than the GPU. This also fails. I also ran the code on a different machine with
a different Intel CPU and it fails there too.

## Tweaks to Ch10_reduction_complete

The two tiny tweaks that I made were as follows.

### Forcing use of CPU instead of GPU

Around line 34, I changed this:

```C
/* Access a device */
err = clGetDeviceIDs(platform, CL_DEVICE_TYPE_GPU, 1, &dev, NULL);
if(err == CL_DEVICE_NOT_FOUND) {
   err = clGetDeviceIDs(platform, CL_DEVICE_TYPE_CPU, 1, &dev, NULL);
}
if(err < 0) {
   perror("Couldn't access any devices");
   exit(1);   
}
```

to this:

```C
/* Access a device */
err = clGetDeviceIDs(platform, CL_DEVICE_TYPE_CPU, 1, &dev, NULL);
if(err < 0) {
   perror("Couldn't access any devices");
   exit(1);   
}
```

### Avoiding CL_OUT_OF_RESOURCES error

The value of `local_size` returned by `clGetDeviceInfo` was 8192.
This caused `clEnqueueNDRangeKernel` to fail with `CL_OUT_OF_RESOURCES`.
I avoided this be hardcoding `local_size` to 128.

Around line 123, I changed this:

```C
err = clGetDeviceInfo(device, CL_DEVICE_MAX_WORK_GROUP_SIZE, 	
      sizeof(local_size), &local_size, NULL);
if(err < 0) {
   perror("Couldn't obtain device information");
   exit(1);   
}
```

to this:

```C
local_size = 128;
```

## Summary

So, in summary, it fails for me on both of these Intel CPUs:

* Intel(R) Core(TM) i7-4720HQ CPU @ 2.60GHz
    * OpenCL 1.2 (Build 57)
    * 5.0.0.57
* Intel(R) Xeon(R) CPU E5-2670 0 @ 2.60GHz
    * OpenCL 1.2 (Build 57)
    * 5.0.0.57

The output it as follows:    

```
$ Ch10_reduction_complete.exe
Global size = 32768
Global size = 256
Global size = 2
Check failed.                       <---
Total time = 684378
```

The line indicated by the arrow shows that the results were not as expected.

## Links

* [OpenCL in Action](https://www.manning.com/books/opencl-in-action)
* [Book's Source Code for VS2010](https://manning-content.s3.amazonaws.com/download/8/56a2ab3-4fe2-440b-8db1-bd5fa93deec6/source_code_vs2010.zip)
* [Intel® SDK for OpenCL™ Applications](https://software.intel.com/en-us/intel-opencl)
