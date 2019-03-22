# Transform Array Operations to Optimize Memory Access for GPGPU

Abstract
========
In this thesis, I investigate the transformation and optimization of memory access patterns, on General Purpose Graphics Processing Units (GPGPU), and its effect on Bohrium. Bohrium is a Just-In-Time (JIT) compiler for NumPy, a Python library for array operations. Bohrium generates code on-the-fly for a GPGPU, which can heavily accelerate computations.

The common goal with Bohrium and this thesis, is to investigate, if it is possible to close the performance gap, between handwritten and auto-generated GPGPU code.


I will implement vector-reductions for the GPGPU, which is currently an unsupported operation. Its addition, will enable large performance gains, as these computations are otherwise carried out by the CPU. It will also allow for operations, that would traditionally use too much memory, as temporary arrays no longer need to be allocated.

I will also deploy methods of optimizing the memory access pattern in general, by rearranging computations to better utilize cache, and achieve memory coalescence. In the same regard, I look at auto-tuning kernel parameters, to find the optimal work-group sizes for the GPGPU. But the implemented optimizations, ends up rendering the auto-tuner redundant.

The changes are tried on a selection of real-world benchmarks collected in the Benchpress repository, which consists of several physics simulations and applications.

Although there is no quantified goal, as to how much faster the benchmarks needs to get, the optimizations have proven to be widely successful. Some benchmarks are running at ten times the original speed, while only a single benchmark is seeing reduced performance.

Comparing performance with a handwritten OpenCL-version of a benchmark, we can show, that the gap between handwritten kernels and Bohrium's auto-generated kernels has significantly narrowed, and is almost comparable. The mentioned benchmark originally took 30.5 seconds to finished in Bohrium. At the end of this thesis, it takes 3 seconds, while the handwritten kernel takes 1.7 seconds.

Conclusion
==========
I started looking at the auto-tuner, as was originally desired for large performance increases. It was planned, that the process should be automated, and made quick enough to find the best kernel parameters on-the-fly. But it did not come to that, as instead, it shed light upon the underlying issues, which made it advantageous. With these issues getting fixed one by one, the auto-tuner got made redundant.


The second topic of this thesis, was vector-reductions. I implemented a method of reducing vector on a GPGPU, where the solution is not trivial. It required extensive knowledge of the hardware, to make it perform well and correctly. But it all paid off. From this functionality alone, we sped up benchmarks, by unforeseen magnitudes. This was also a turn for Bohrium, as this was the first operation to not conform with the regular access patterns of supported operations.


As the auto-tuner had revealed some issues with the generated kernels, I moved on to fix these, to get a consistent result from the auto-tuner. This led to the discovery of access patterns going against the rules of coalesced memory access, which is preferred by the GPGPU hardware. After correcting these patterns, we saw staggering improvements in several benchmarks. The parameters of the auto-tuner also now made more sense, as they became less erratic. But the changes also came, with its drawbacks. Although the new pattern is more correct in the theoretical sense, one benchmark seems to have benefited from the wrong access pattern.


After changing the access pattern, I identified, that the benchmark, was now performing a reduction, in an uncoalesced pattern. Which came to light, as two wrongs did make a right, when it came to access patterns. As both the kernel parameters and the reduction were transposed compared to the optimal, they became right in their own sense. With the changed access pattern, just the reduction is left transposed.


To get this sorted out, I implemented a special type of reductions. The new segmented reductions, take over the regular access pattern of reductions in inner ranks, and make their access pattern become coalesced. The new feature did work, as it brought down the execution-time down to bearable levels. But it still came with a penalty, compared to the original state of Bohrium.


Before concluding the thesis, I wanted to proof, that the changes made, has not broken anything in Bohrium. I did this by running my code through Bohrium's own tests for correctness, and additionally implement my own. After correcting a few mistakes, I finally consider the code stable, and ready for other developers to take over.


All in all, this thesis has introduced incredible improvements to Bohrium. Most of the selected benchmarks see performance increases in the range of 60-90% shorter execution-time. The improvements I have made in this thesis, has even made these increase come, without the cost of an auto-tuner. The speed-up appear because of better control of the kernel, and better understanding of the hardware.


The improvements even carry over to other hardware, which signifies, that this is not a over-fitted optimization of the hardware I used, but a general improvement.

It is always hard to make unconditional optimizations, especially, when we are not in control of the input. But the balance between performance gains and losses, could be seen as acceptable, if the selection of benchmarks, are representative of typical workloads.

In the end, the changes has positively affected Bohrium, by lowering the execution-time. And in the big picture, this also means, that we have narrowed the gap between developing native OpenCL code, and regular Python code. The more this gap closes, the less appealing it is, to spend the time developing OpenCL kernels, when almost the same performance, can be achieved in a higher-level language.


Reflection
==========
The exact implementations, I have produced in this thesis, are very narrowly dedicated to Bohrium, and would not just plug into any OpenCL project. In the sense, that the code is centered around Bohrium's internal representation, and how each component communicates and operates. This does not include the vector-reduction implementation, which only requires including the header file, and calling the functions to be used in any OpenCL kernel.

But on a theoretical level, all the changes are widely usable.

Changing the access pattern, from the most obvious way to parallelize, to the way that maximizes coalesced memory access, is important. And this thesis demonstrates, that there are large performance gains to get from even small changes to the code and parameters. The changes, does not even need to modify that many lines of code. It is a lot harder to come up with the solution, than to actually go through with the implementation.

Even though it did not change performance as much, one of the important observations I made, were that of transposing reductions. Applying this transposition to every reduction, gives an incredible advantage, when generating or handwriting kernels. The rather simple technique, simplifies kernel generation and kernel fusion. By always reducing the inner-most axis, it becomes a lot simpler to handle memory dependencies, and it allows to fuse reductions together, which would otherwise be incompatible. And most importantly, it also allows to place the reductions as deep as possible, to not sacrifice the potential parallelism of a kernel.

Anyone working with reduction of tensors, would likely benefit from this technique. And even though this implementation only applies to Bohrium, it is a very common operation, which has a very simple solution. It can also be used to place nested reductions in any favorable order, meaning, we can in some cases, fully avoid uncoalesced memory access, without complicating the generated kernel.

These findings and solutions will apply anywhere, and could easily be implemented in other kernel generating software, as well as handwritten kernels. In fact, the paradigm of these kernels, all follow the same map-reduce type, which is very common in parallel programming.
