---
layout: article
title: Debugging True Crime on the PS2
author: PSI
---

True Crime uses a custom streaming IOP module that acts as a wrapper around the CDVD manager. It allows the game to asynchronously stream disc data from multiple sources. On dobiestation, the game's stdout complains about not being able to start a streaming thread while loading assets. I figured that there's an issue with the streaming module because of this, so I opened up Ghidra and worked on reversing the driver on the EE and the module itself on the IOP.

Thankfully, the IOP module has full debug symbols, which made my job a little easier. Reversing that and the other functions in the game took a couple of days, but I didn't find anything wrong immediately, so I decided to look at the code that was crashing on the TLB miss. I found out that it had something to do with initializing a 3D model. The header for this model was all zeroes, indicating that the model was not loaded into memory.

Normally when the game loads a file, it'll open a stream, read the data, then close the stream. However, during initialization, the game opens three streams related to loading world data and never closes them. The game was trying to use one of those streams to load the model, but that stream had been deleted. Further investigation revealed that the game called the stream module initialization function more than once, and this was causing the stream queue to be reset. But... the initialization function is programmed to not do anything when it has already initialized the module. After it initializes the module, the function sets a variable to 1. But something was changing this variable back to 0.

I modified my emulator to print a debug statement every time that variable was modified. The thing that changed the variable back to 0 was... the IOP's reply sent over DMA. Suddenly, it made sense. The game was relying on undefined behavior. The driver on the EE side allocates a 4-byte buffer used to hold the IOP's reply, and the is\_initialized variable is right next to this buffer in memory. However, the EE DMAC must transfer in units of quadwords, or 16 bytes, at minimum. So although the driver told the IOP that the buffer is only 4 bytes long, the IOP has to send 16 bytes back to the EE. The IOP only sets the first 4 bytes, meaning that the other 12 bytes are garbage. Because of this, the is\_initialized variable is overwritten with whatever garbage is next to the IOP's reply buffer in memory. As it turns out, the garbage memory is set to 0 when the module is initialized and never set to any other value.

So to summarize, UB in the EE driver causes the game to think the module has not been initialized, resetting three world data streams and making the game unable to load models off the disc

As for the patch, the streaming module has a feature where it can queue up commands to send to the IOP, instead of sending the commands each time. 
