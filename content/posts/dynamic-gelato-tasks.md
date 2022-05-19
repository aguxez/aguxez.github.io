---
title: "Dynamic Gelato Tasks"
date: 2022-05-18T20:54:10-05:00
draft: false
---

I've been playing around with some smart contract automation through [Gelato](https://app.gelato.network/). The integration is pretty straightforward,
at [ZED](https://zed.run/) we've been using it to automate some parts of our system in a reliable and decentralized way. One normal use case for this type of jobs
is to iterate over a queue until you have an item that meets a certain criteria.
You can build a mechanism that works as a FIFO queue very easily with OZ's `EnumerableSet`, for example. I want to talk about 
a very obvious problem with implementing these type of systems on a blockchain like Ethereum (or EVM-based). These
jobs will be, most of the time, slow. Why? Because of the nature of the exection. Txs, unless
handled in a concurrent way, will be working one by one. Don't expect 10s of txs being executed at the same time.

One option, specifically for **Gelato** is to dynamically spawn tasks (same as you'd do in OTP😉) for each request
that you perform instead of adding it to the end of a queue, here's an example (with a Gist at the bottom)

```solidity
function createTask() external {
    IOps(ops).createTaskNoPrepayment(
        address(this),
        this.removeTask.selector,
        address(this),
        abi.encodeWithSelector(this.checker.selector, block.number),
        ETH
    );
    
    tsToTask[block.number] = taskId;
}
```
In this snippet we're just creating a function to create a task on the `ops` contract, the resolver function is 
`checker` in our contract, it will also be paid from the contract's balance. The typespec of this function is

```solidity
function createTaskNoPrepayment(
    address execAddress, 
    bytes4 execSelector, 
    address resolver, 
    bytes calldata resolverData, 
    address feeToken
) external returns (bytes32 taskId);
```

As you can see, this function returns a task ID which you can manipulate in any way you'd like. This is your 
reference to cancel a task after you're done with the job because we're good citizens. 

The resolver, as said above, is `checker` which is a function that should return `bool canExec, bytes memory execPayload`.

The function we _should_ use to for the executor is `removeTask`, in this example.

```solidity
function checker(uint256 ts) external view returns (bool canExec, bytes memory execPayload) {
    // `ts` is the `block.number` we passed above
    if (tsToTask[ts] != 0) {
        canExec = true;
        
        execPayload = abi.encodeWithSelector(this.removeTask.selector, ts)
    }
}

function removeTask(uint256 ts) external {
    bytes32 taskId = tsToTask[ts];
    
    require(taskId != 0, "Task: task already execuetd");
    
    // ... some code to pay ops executors
    
    IOps(ops).cancelTask(taskId);
    
    delete tsToTask[ts];
}
```

This way you have a contract that will dynamically spawn Gelato (🍦) tasks and cancel them after they're done. 
The only _issue_ here is that there's a small delay when you create a task before it starts executing, it takes
near a minute. For small activity tasks this will actually slow things down, but for something like 5-10 requests
happening at the same time, it will increase throughput. It's better to make each wait 3-4 minutes than waiting
10-20 minutes for a small sized queue to execute. 

[Here's a Gist with a fully working example](https://gist.github.com/aguxez/b92b6ae2d9e8959a3856c7c340b4142c)