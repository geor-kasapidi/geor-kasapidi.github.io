---
layout: post
title:  "Improve Metal GPU utilization"
date:   2022-08-17 09:00:00 +0300
categories: metal swift
keywords: swift, metal, ios, command buffer, mtlcommandbuffer, mpscommandbuffer, gpu, programming
---
Building a fast and efficient CPU<->GPU pipeline is not a trivial engineering task. Since the whole point of using a GPU in your computing is speed, you want to find the best way to use it. And the most obvious idea is to parallelize calculations between CPU and GPU. In this article, i want to show you a cool trick available since iOS 13 that will allow you to optimize your GPU usage.

<!-- more -->

But before we get into the tips and tricks section, let's revisit standard Metal flow and hightlight the bottleneck.

## Standard Metal flow

The entry point for interaction with the GPU in metal is *device* object (`MTLDevice`), which creates a *command queue* (`MTLCommandQueue`). And the command queue creates *command buffers* (`MTLCommandBuffer`):

![metal_flow](/assets/images/mpscommandbuffer/metal_flow.svg)

You create one command queue instance and use it across your application:

{% highlight swift %}
guard let device = MTLCreateSystemDefaultDevice(),
      let commandQueue = device.makeCommandQueue()
else {
    fatalError()
}
{% endhighlight %}

And use that instance wherever you need GPU access, creating command buffers:

{% highlight swift %}
let commandBuffer = commandQueue.makeCommandBuffer()!
// command encoding
commandBuffer.commit()
commandBuffer.waitUntilCompleted()
{% endhighlight %}

In this code, the following happens: we encode commands into the buffer (it is very important to understand that at this point there is no real execution yet), then we commit the command buffer (send commands to the GPU) and wait for the GPU to complete its work. Thus, mixing CPU and GPU calculations in the code, we can see the following picture:

![cpu_gpu_standard_flow](/assets/images/mpscommandbuffer/cpu_gpu_standard_flow.svg)

Dotted line means `commit` + `waitUntilCompleted` calls.

The problem here is the lack of parallelism: the CPU waits for the GPU, and the GPU, in turn, does nothing while the CPU encodes commands.

To avoid idle processors, we will use a well-known approach.

## Two-stage pipeline

or double buffering:

![cpu_gpu_double_buffering](/assets/images/mpscommandbuffer/cpu_gpu_double_buffering.svg)

Dotted line means only the `commit` call of the command buffer. This call is non-blocking, so CPU can continue to encode new commands, while GPU is executing current ones. Thus, we have divided our command buffer into several, and this allows us to use our processors much more efficiently. As i said, this approach is well known, but has some limitations:

1. Your code gets more complex - you need to manage N command buffers instead of 1.
2. You cannot use temporary MPS objects (`MPSTemporaryImage`, etc.) because they are alive as long as their parent command buffer lives.

The second point is an advanced feature, but very important if you are deep into metal programming, especially with MPS framework (metal performance shaders).

*Question*: How to avoid these limitations, but keep the benefits of double buffering?

*Answer*: Use [MPSCommandBuffer](https://developer.apple.com/documentation/metalperformanceshaders/mpscommandbuffer).

## MPSCommandBuffer

If you're familiar with Metal framework, you might know that most entities like `MTLDevice` and `MTLCommandQueue` are protocols. `MTLCommandBuffer` is also a protocol. `MPSCommandBuffer` conforms to it, adding some extra features. The most useful of them is the `commitAndContinue` method. `MPSCommandBuffer` internally recreates the actual command buffer when you call `commitAndContinue` and ensures that any temporary objects remain valid after the command buffer is recreated.

{:refdef: style="text-align: center;"}
![mps_command_buffer](/assets/images/mpscommandbuffer/mps_command_buffer.svg)
{:refdef}

Nothing changes for you as a developer, except for the additional method, you write the same command encoding code, replacing `let commandBuffer = commandQueue.makeCommandBuffer()!` with `let commandBuffer = MPSCommandBuffer(from: commandQueue)`. And every time you call `commitAndContinue` during command encoding, your GPU receives the next batch of commands:

{% highlight swift %}
let commandBuffer = MPSCommandBuffer(from: commandQueue)
// command encoding
commandBuffer.commitAndContinue()
// command encoding
commandBuffer.commit()
commandBuffer.waitUntilCompleted()
{% endhighlight %}

Feel free to use `MPSCommandBuffer` with any metal code, it's not limited to metal performance shaders. This API is especially useful, when your CPU part is heavy.

## Bonus

A small code snippet:

{% highlight swift %}
extension MTLCommandQueue {
    @discardableResult
    func sync<T>(_ body: (MPSCommandBuffer) throws -> T) rethrows -> T {
        let commandBuffer = MPSCommandBuffer(from: self)

        let result = try autoreleasepool {
            try body(commandBuffer)
        }

        commandBuffer.commit()
        commandBuffer.waitUntilCompleted()

        return result
    }
}
{% endhighlight %}

{% highlight swift %}
let result = commandQueue.sync { commandBuffer in
    ...
    commandBuffer.commitAndContinue()
    ...
    commandBuffer.commitAndContinue()
    return ...
}
{% endhighlight %}

Thanks for reading!
