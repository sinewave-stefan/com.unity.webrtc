target_sources(
  WebRTCLib
  PRIVATE GraphicsDevice.cpp
          GraphicsDevice.h
          GraphicsUtility.cpp
          GraphicsUtility.h
          IGraphicsDevice.h
          ITexture2D.h
          ScopedGraphicsDeviceLock.cpp
          ScopedGraphicsDeviceLock.h)

if(Windows)
  if(Vulkan_FOUND)
    add_subdirectory(Vulkan)
  endif()
  add_subdirectory(D3D11)
  add_subdirectory(D3D12)
  if(CUDA_FOUND)
    add_subdirectory(Cuda)
  endif()
elseif(iOS OR macOS)
  add_subdirectory(Metal)
elseif(Linux)
  add_subdirectory(Vulkan)
  add_subdirectory(OpenGL)
  add_subdirectory(Cuda)
elseif(Android)
  add_subdirectory(Vulkan)
  add_subdirectory(OpenGL)
endif()
