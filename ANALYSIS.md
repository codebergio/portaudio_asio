# Deep Analysis of Channel Mapping Issue

## The Problem
Sound outputs to channels 1-2 instead of requested channels 3-4, despite code modifications.

## Root Cause Analysis

### The Flow:
1. **OpenStream()** - Setup phase
   - We set `outputChannelSelectors[i] = 2 + i` (so {2, 3} for 2 output channels)
   - We set `asioBufferInfos[inputChannelCount + i].channelNum = outputChannelSelectors[i]`
   
2. **ASIOCreateBuffers()** - Driver allocation phase
   - Driver is called with asioBufferInfos[] array with our channel numbers
   - **CRITICAL QUESTION**: Does ASIO driver respect our channelNum requests?

3. **Buffer Pointer Setup** - (lines 2408-2428)
   ```cpp
   for( int i=0; i<outputChannelCount; ++i ) {
       int asioIndex = inputChannelCount + i;  // WRONG!
       stream->outputBufferPtrs[0][i] = stream->asioBufferInfos[asioIndex].buffers[0];
       stream->outputBufferPtrs[1][i] = stream->asioBufferInfos[asioIndex].buffers[1];
   }
   ```
   This copies buffers from asioBufferInfos[inputChannelCount + i]
   
4. **Audio Callback** - (lines 3184-3185)
   ```cpp
   for( int i=0; i<outputChannelCount; ++i )
       PaUtil_SetNonInterleavedOutputChannel(..., i, outputBufferPtrs[index][i]);
   ```

## The Critical Issue

**We're making a fundamental assumption that's likely wrong:**

When we call ASIOCreateBuffers(), the ASIO driver likely allocates buffers **sequentially**:
- It creates buffer for physical channel 0
- It creates buffer for physical channel 1  
- It creates buffer for physical channel 2
- It creates buffer for physical channel 3
- etc.

The buffers are returned in asioBufferInfos[] in the order they were requested.

**Our current code does:**
1. asioBufferInfos[inputCount + 0].channelNum = 2 (request channel 3)
2. asioBufferInfos[inputCount + 1].channelNum = 3 (request channel 4)
3. Call ASIOCreateBuffers()
4. Get back buffers - but probably for channels 2 and 3, not in our mapping order!
5. Assign outputBufferPtrs[0][0] = asioBufferInfos[inputCount + 0].buffers (whatever channel that is)
6. Assign outputBufferPtrs[0][1] = asioBufferInfos[inputCount + 1].buffers (whatever channel that is)

### The Solution

We need to populate asioBufferInfos[] in the order that ASIOCreateBuffers expects:
- Set asioBufferInfos[inputCount + 0] to request channel 0
- Set asioBufferInfos[inputCount + 1] to request channel 1
- ... for ALL channels that the ASIO driver has
- Call ASIOCreateBuffers()
- Then only use the buffers we wanted (for channels 2, 3)

OR

We need to reorder the buffer pointer assignments based on which channel each asioBufferInfo actually corresponds to.

## Recommended Fix

After ASIOCreateBuffers(), we should:
1. Check what channelNum was actually assigned to each asioBufferInfos entry
2. Build a mapping: user_channel_index -> asio_buffer_index
3. Use this mapping when setting up outputBufferPtrs

Example:
```cpp
// Create mapping from desired output channel to actual ASIO buffer index
int *asioBufferIndexForOutputChannel = malloc(outputChannelCount * sizeof(int));

for( int i=0; i<(inputChannelCount + outputChannelCount); ++i ) {
    if( !stream->asioBufferInfos[i].isInput ) {
        int asioPhysicalChannel = stream->asioBufferInfos[i].channelNum;
        // Map this buffer index to the correct user channel
        for( int j=0; j<outputChannelCount; ++j ) {
            if( outputChannelSelectors[j] == asioPhysicalChannel ) {
                asioBufferIndexForOutputChannel[j] = i;
            }
        }
    }
}

// Now use the mapping
for( int i=0; i<outputChannelCount; ++i ) {
    int correctAsioIndex = asioBufferIndexForOutputChannel[i];
    stream->outputBufferPtrs[0][i] = stream->asioBufferInfos[correctAsioIndex].buffers[0];
    stream->outputBufferPtrs[1][i] = stream->asioBufferInfos[correctAsioIndex].buffers[1];
}
```

## Alternative: Request All Channels

Instead of requesting only 2 channels (0 and 1) and remapping, we could:
1. Request 4 channels (0, 1, 2, 3) from the ASIO driver
2. Use channels 2 and 3 for output
3. Discard channels 0 and 1

But this is wasteful and might break with different numbers of output channels.
