1786382118
directx12
DirectX12 시작하기 - 1

### Factory
후술할 DirectX의 요소들을 생성하기 위해서는
Factory의 생성이 필요하다.
Factory는 Adapter 등을 생성하는 데에 필요하다.

```cpp
Factory& Factory::get() {
  static Factory factory;
  if (factory._factory != nullptr) { return factory; }

  ComPtr<IDXGIFactory4> factory4;
  UINT createFactoryFlags = 0;
#if defined(_DEBUG)
  createFactoryFlags = DXGI_CREATE_FACTORY_DEBUG;
#endif
  throwOnFail(CreateDXGIFactory2(createFactoryFlags, IID_PPV_ARGS(&factory4)));
  
  factory._factory = factory4;
  
  return factory;
}
```


### Adapter
Adapter는 Device를 만드는 데에 필요하다.

```cpp
Adapter& Adapter::get() {
  static Adapter adapter;
  if (adapter._adapter != nullptr) { return adapter; }

  auto& factory = Factory::get();

  Microsoft::WRL::ComPtr<IDXGIAdapter1> adapter1;
  Microsoft::WRL::ComPtr<IDXGIAdapter4> adapter4;

  if (_isWarpEnabled) {
    throwOnFail(factory.d3d12()->EnumWarpAdapter(IID_PPV_ARGS(&adapter1)));
    throwOnFail(adapter1.As(&adapter4));
  } else {
    SIZE_T maxDedicatedVideoMemory = 0;
    for (UINT i = 0; factory.d3d12()->EnumAdapters1(i, &adapter1) != DXGI_ERROR_NOT_FOUND; ++i) {
      DXGI_ADAPTER_DESC1 adapterDesc1;
      adapter1->GetDesc1(&adapterDesc1);

      if (
        (adapterDesc1.Flags & DXGI_ADAPTER_FLAG_SOFTWARE) == 0 &&
        SUCCEEDED(D3D12CreateDevice(adapter1.Get(), D3D_FEATURE_LEVEL_11_0, __uuidof(ID3D12Device), nullptr)) &&
        adapterDesc1.DedicatedVideoMemory > maxDedicatedVideoMemory
      ) {
        maxDedicatedVideoMemory = adapterDesc1.DedicatedVideoMemory;
        throwOnFail(adapter1.As(&adapter4));
      }
    }
  }

  adapter._adapter = adapter4;
  
  return adapter;
}
```

### Device
Device는 CommandQueue 등을 만들기 위해 필요하다.

```cpp
Device& Device::get() {
  static Device device; 
  if (device._device != nullptr) { return device; }

  auto& adapter = Adapter::get();

  Microsoft::WRL::ComPtr<ID3D12Device2> d3d12Device2;
  throwOnFail(D3D12CreateDevice(adapter.d3d12().Get(), D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&d3d12Device2)));
  
#if defined(_DEBUG)
  Microsoft::WRL::ComPtr<ID3D12InfoQueue> pInfoQueue;
  
  if (SUCCEEDED(d3d12Device2.As(&pInfoQueue))) {
    pInfoQueue->SetBreakOnSeverity(D3D12_MESSAGE_SEVERITY_CORRUPTION, TRUE);
    pInfoQueue->SetBreakOnSeverity(D3D12_MESSAGE_SEVERITY_ERROR, TRUE);
    pInfoQueue->SetBreakOnSeverity(D3D12_MESSAGE_SEVERITY_WARNING, TRUE);

    D3D12_MESSAGE_SEVERITY Severities[] = { D3D12_MESSAGE_SEVERITY_INFO };

    D3D12_MESSAGE_ID DenyIds[] = {
      D3D12_MESSAGE_ID_CLEARRENDERTARGETVIEW_MISMATCHINGCLEARVALUE,
      D3D12_MESSAGE_ID_MAP_INVALID_NULLRANGE,
      D3D12_MESSAGE_ID_UNMAP_INVALID_NULLRANGE,
    };

    D3D12_INFO_QUEUE_FILTER NewFilter = {};
    NewFilter.DenyList.NumSeverities = _countof(Severities);
    NewFilter.DenyList.pSeverityList = Severities;
    NewFilter.DenyList.NumIDs = _countof(DenyIds);
    NewFilter.DenyList.pIDList = DenyIds;

    throwOnFail(pInfoQueue->PushStorageFilter(&NewFilter));
  }
#endif

  device._device = d3d12Device2;

  return device;
}
```

