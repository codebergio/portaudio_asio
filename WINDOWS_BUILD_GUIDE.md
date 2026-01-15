# Windows DLL Compilation Guide

## Prerequisites

- Visual Studio 2022 (or 2019+)
- Git
- CMake 3.20+

## Build Steps for Windows

### Option 1: Using CMake GUI

1. Clone the repository:
   ```powershell
   git clone https://github.com/yourusername/portaudio_asio.git
   cd portaudio_asio
   git submodule update --init portaudio
   ```

2. Open CMake GUI
   - Source: `C:\...\portaudio_asio\portaudio`
   - Build: `C:\...\portaudio_asio\portaudio\build`

3. Configure with:
   - Generator: `Visual Studio 17 2022`
   - Optional platform: `x64`
   - Enable: `PA_USE_ASIO=ON`
   - Set CMAKE_PREFIX_PATH to: `C:\...\portaudio_asio\asio` (absolute path)

4. Generate and Open Project

5. In Visual Studio:
   - Right-click `portaudio` solution
   - Build Solution (or Build | Release)

6. Find the DLL at: `build\Release\portaudio.dll`

### Option 2: Command Line

```powershell
cd portaudio
rm build -r -force
mkdir build
cd build

cmake .. -G "Visual Studio 17 2022" -A x64 `
  -DPA_USE_ASIO=ON `
  -DCMAKE_PREFIX_PATH="C:\...\portaudio_asio\asio"

cmake --build . --config Release
```

DLL will be at: `build\Release\portaudio.dll`

### Option 3: GitHub Actions (Automated)

The repository includes `.github/workflows/build-windows.yml` which automatically:
1. Checks out the code
2. Initializes the portaudio submodule
3. Configures CMake with ASIO support
4. Builds with Visual Studio 2022
5. Uploads the DLL as an artifact

To use:
1. Push to your repository
2. Go to Actions tab
3. Click "Build Windows DLL"
4. Download artifact after build completes

## Verifying the Build

After build, verify the DLL has ASIO support:

### Using objdump (Linux/WSL)
```bash
objdump -t build/portaudio.dll | grep -i asio
```

### Checking symbol presence
The DLL should contain symbols like:
- `Pa_GetAsioVersion`
- `Pa_OpenStream` (should work with ASIO streams)
- Internal ASIO callbacks

## Using the Compiled DLL

1. Copy `portaudio.dll` to your application's directory or PATH

2. In your code:
   ```c
   #include "portaudio.h"
   
   // When opening stream, specify output device as ASIO device
   // The modified code will automatically redirect channels 3-4
   Pa_OpenStream(&stream, NULL, &outputParameters, sampleRate, 
                 framesPerBuffer, paClipOff, callback, userData);
   ```

3. To see debug output on Windows:
   - Run application in a debugger
   - Or redirect stderr to see PA_DEBUG messages

## Troubleshooting Windows Build

### Issue: "Visual Studio 17 2022 Generator not found"
- Solution: Install Visual Studio 2022 Community (free)
- Or use: `-G "Visual Studio 16 2019"` for VS 2019

### Issue: ASIO SDK not found
- Ensure `asio/` directory exists in repository root
- Check CMAKE_PREFIX_PATH is set correctly
- Run: `cmake . -DCMAKE_PREFIX_PATH="C:\full\path\to\asio"`

### Issue: Build fails with undefined references
- Clean and rebuild: `rm build -r -force && mkdir build`
- Verify ASIO headers are present in `asio/` subdirectory
- Check that `.gitmodules` correctly references ASIO

## Performance Tuning

For best results:
- Use Release build (not Debug) for production
- Ensure buffer sizes are 256 or 512 frames for low latency
- Monitor PA_DEBUG output for any buffer underruns

## Next Steps

1. Test the DLL with your application
2. Monitor debug output to verify channel mapping
3. Check audio comes from correct interface outputs
4. Adjust hardcoded channel numbers if needed (in `pa_asio.cpp` line ~2121)
