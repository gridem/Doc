# Hot Update Ideas

## Replacing The Function or Methods

Each method of function must have the gap at the beginning e.g. using assembler NOP instructions. We would like to change one function to another function. For that purpose we perform the following steps:

1. Add jump to the new function at the beginning of the old function atomically by using assembler jump instruction. To perform this step atomically we need to introduce gap before function and small gap in the beginning function. Before function we add long jump while at the beginning we add short jump. Short jump allows to do it atomically.
2. Wait for some time.
3. Replace all callers addresses from old to new one. Here we can do the following: instead of directly call the procedure we call the stab that contains only jump to necessary address. We replace only single place and all callers automatically invokes those addresses.
4. Unload unused functions from the process.

## Late Binding

Another idea could be taken from Alan Key:

> OOP to me means only messaging, local retention and protection and hiding of state-process, and extreme late-binding of all things.

Late binding allows to change components without significant efforts. Microservices just uses that technology and they can be updated separately.

## Data and Functionality

To simplify with update approach we need to split the update on 2 phases:

1. Data update.
2. Logic/functionality update.

The example is router: either logic or data can be updated on the fly, but not both.

### Smooth Data Update

The idea is based on the following fact. Small object can be updated easily by reconstructing object from the scratch by using snapshot.

Nevertheless big objects cannot be recreated instantaneously. The reason that they may occupy a lot of space that must be loaded from the disk or transferred from the network or from the process. And the following idea is appeared: to use iterative approach.

Big object is split by chunks. Each chunk is updated sequentially. The old process must readdress original requests from client if the client obtains the object that has been transferred to the new process.

## Class Update

Class consists of data and methods. Simultaneous update of both is the complex task and cannot be done in a simple manner for the most generic case.

The idea is based on erlang model: where actor resends the current context (data) to the new actor that contains new functionality. Thus actor represents the methods while message represents the data.

In order to map erlang model one may consider the following approach: add special constructor that accepts old data (or old class) and generates appropriate instance for new class.
