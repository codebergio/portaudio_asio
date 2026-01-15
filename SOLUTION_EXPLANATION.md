# ASIO Channel Redirection - Deep Dive Analysis & Solution

## Problem Statement
PortAudio was outputting audio to ASIO channels 0-1 (displayed as channels 1-2 in user's audio interface) instead of the requested channels 2-3 (displayed as 3-4), despite code modifications to redirect the output.

## Root Cause

The issue had **two layers of complexity**:

### Layer 1: How ASIO Buffer Allocation Works
When you call `ASIOCreateBuffers()`, the ASIO driver:
1. Receives an array of `ASIOBufferInfo` structures
2. Each structure has:
   - `isInput` - whether it's input or output
   - `channelNum` - which physical channel (0, 1, 2, 3...)
   - `buffers[2]` - pointers to the allocated buffers
3. The driver populates the `buffers[]` arrays with allocated memory
4. Returns the fully populated structures

### Layer 2: The PortAudio Bug
After calling `ASIOCreateBuffers()`, PortAudio was assigning buffer pointers like this:

```cpp
for( int i=0; i<outputChannelCount; ++i ) {
    int asioIndex = inputChannelCount + i;  // WRONG!
    stream->outputBufferPtrs[0][i] = stream->asioBufferInfos[asioIndex].buffers[0];
    stream->outputBufferPtrs[1][i] = stream->asioBufferInfos[asioIndex].buffers[1];
}
```

**The problem:** This assumes a direct 1:1 mapping between:
- User's output channel `i` 
- ASIO buffer info array index `inputChannelCount + i`

But when you specify `channelNum = 2` for the first output channel, that doesn't guarantee the buffer will be at index `inputChannelCount + 0`!

### What Actually Happens
Scenario: User wants to output to ASIO channels 2 and 3 (third and fourth physical channels)

**What we did:**
```cpp
asioBufferInfos[inputChannelCount + 0].channelNum = 2;
asioBufferInfos[inputChannelCount + 1].channelNum = 3;
ASIOCreateBuffers(...);  // Driver allocates
```

**What we assumed would happen:**
```
outputBufferPtrs[0][0] = asioBufferInfos[inputChannelCount + 0].buffers[0]  // Channel 2 ✓
outputBufferPtrs[0][1] = asioBufferInfos[inputChannelCount + 1].buffers[0]  // Channel 3 ✓
```

**What actually happened:**
The ASIO driver might allocate buffers in ANY order it chooses. The actual `channelNum` assignments after `ASIOCreateBuffers()` might not match our expectations.

## The Solution

The fix is to **search for the correct buffer index** instead of assuming a direct mapping:

```cpp
/* For each desired output channel, find its ASIO buffer index */
for( int desiredChanIdx = 0; desiredChanIdx < outputChannelCount; ++desiredChanIdx ) {
    int desiredPhysicalChannel = outputChannelSelectors[desiredChanIdx];
    
    /* Search through all ASIO buffers to find the one with this physical channel */
    for( int asioIdx = inputChannelCount; asioIdx < (inputChannelCount + outputChannelCount); ++asioIdx ) {
        if( stream->asioBufferInfos[asioIdx].channelNum == desiredPhysicalChannel ) {
            foundAsioIndex = asioIdx;
            break;
        }
    }
    
    // Use foundAsioIndex to get the correct buffer
    stream->outputBufferPtrs[0][desiredChanIdx] = stream->asioBufferInfos[foundAsioIndex].buffers[0];
}
```

This way, we're guaranteed to use the buffers that actually correspond to the channels we requested.

## Changes Made

### In `portaudio/src/hostapi/asio/pa_asio.cpp`:

1. **Enhanced debug logging** at key points:
   - After `ASIOCreateBuffers()`: Dump all asioBufferInfos to see what was allocated
   - After `ASIOGetChannelInfo()`: Show channel info for each allocated buffer
   - During output buffer mapping: Trace the search and final assignments

2. **Critical buffer mapping fix**:
   - Allocate `asioBufferIndexForOutputChannel[]` to store the mapping
   - For each desired output channel, search `asioBufferInfos` to find matching `channelNum`
   - Use the found index when assigning `outputBufferPtrs`
   - Fallback to sequential mapping if no match found (for user-provided selectors)

3. **Hardcoded channel selection**:
   - Still hardcoded to redirect channels 3-4 (indices 2-3) in the selector initialization

## Debug Output

With these changes, when opening a stream, you'll see:

```
OpenStream: Setting outputChannelSelectors[0] = 2 (channel 3)
OpenStream: Setting outputChannelSelectors[1] = 3 (channel 4)
Forcing output to channels starting from 3

=== AFTER ASIOCreateBuffers ===
asioBufferInfos[0]: INPUT, channelNum=0, buffers=(0x..., 0x...)
asioBufferInfos[1]: INPUT, channelNum=1, buffers=(0x..., 0x...)
asioBufferInfos[2]: OUTPUT, channelNum=2, buffers=(0x..., 0x...)
asioBufferInfos[3]: OUTPUT, channelNum=3, buffers=(0x..., 0x...)

ASIOGetChannelInfo[2]: channel=2, OUTPUT, type=0, name="Out 3"
ASIOGetChannelInfo[3]: channel=3, OUTPUT, type=0, name="Out 4"

=== Building output channel mapping ===
Output channel 0 (phys 2) -> ASIO buffer index 2
Output channel 1 (phys 3) -> ASIO buffer index 3

Output buffer assignment: user_channel=0, asioIndex=2, channelNum=2, buffers=(0x..., 0x...)
Output buffer assignment: user_channel=1, asioIndex=3, channelNum=3, buffers=(0x..., 0x...)
```

This gives you complete visibility into:
1. What physical channels were requested
2. How the ASIO driver allocated them
3. Which buffer pointers are being used for each user channel

## How to Test

1. Rebuild with these changes:
   ```bash
   cd portaudio
   rm -rf build && mkdir build && cd build
   cmake .. -G "Unix Makefiles"
   make -j4
   ```

2. Use the compiled library with your application
3. Enable debug output to see the mapping
4. Test with a mono or stereo application outputting to your Soundcraft EVO 4

## Expected Result

Audio should now output to channels 3-4 (indices 2-3) on the ASIO device instead of channels 1-2 (indices 0-1).

## Files Modified

- `portaudio/src/hostapi/asio/pa_asio.cpp` - Buffer mapping and debug logging
- `.github/workflows/build-windows.yml` - Configured for Windows DLL builds (no changes this session)
