---
layout: post
title:  "Tomb Raider Reboot"
date:   2026-08-27 14:49:04 +0200
categories: jekyll update
---

# Running TR2_x64_release.exe

Fetch the executable from [here](https://debugging.games/_files/Windows/[WIN]%20Rise%20of%20the%20Tomb%20Raider%20[2021-10-22]%20(PDB).7z)

First patch the executable. Edit the HasValidMainUser function (offset = 0xe9bc40) to always return true. 

![Image](/assets/TombRaider/images/HasValidMainUserAsm.png)

Then run the executable with the following arguments: `-archive -norootchange -mainmenu -noassert -launcher`

# Injecting an ImGUI Window into ROTTR

To inject our window we create a dxgi.dll that acts as a middle man between the game and the real dxgi.dll. Our dxgi.dll has a modified version of `CreateDXGIFactory` and `CreateDXGIFactory1` that replace the virtual `CreateSwapChain` function in the vtable with our own implementation then calls the real `CreateSwapChain` function

{% highlight cpp %}
HRESULT WINAPI CreateDXGIFactory1(REFIID riid, _COM_Outptr_ void** ppFactory) {
    HMODULE dxgi_lib = LoadLibraryA("C:\\Windows\\System32\\dxgi.dll");
    CreateDXGIFactory_t function = (CreateDXGIFactory_t)GetProcAddress(dxgi_lib, "CreateDXGIFactory1");
    HRESULT result = function(riid, ppFactory);
    IDXGIFactory1** factory = reinterpret_cast<IDXGIFactory1**>(ppFactory);

    void** vtable = *(void***)(*factory);

    if (oCreateSwapChain1 == nullptr) {
        oCreateSwapChain1 = (CreateSwapChain_t)vtable[10];
    }

    DWORD old_protection;
    VirtualProtect(&vtable[10], 8, PAGE_READWRITE, &old_protection);
    vtable[10] = (void*)CreateSwapChain1;
    VirtualProtect(&vtable[10], 8, old_protection, &old_protection);

    return result;
}
{% endhighlight %}

We are first dynamically loading the real dxgi.dll and retrieving the real `CreateSwapChain` function and calling it. We then remove the write protection from the vtable (since it is part of the excutable and has write protection) and replace the entry at index 10 which is the CreateSwapChain with our own implementation.

Our implementation of `CreateSwapChain` is initializes ImGUI with the swap chain that has been created and replaces the `Present` function with our own implementation that renders the ImGUI window every frame.

{% highlight cpp %}
HRESULT __fastcall hkPresent(IDXGISwapChain* pThis, UINT SyncInterval, UINT Flags) // 000002DC26616710
{
    GUI::StartFrame();
    ImGui::DockSpaceOverViewport(0, ImGui::GetMainViewport(), ImGuiDockNodeFlags_PassthruCentralNode);
    ImGui::Begin("Asil's Debug Menu");
    ImGui::Text("Hello, Lara Croft!");
    ImGui::End();
    GUI::EndFrame();
    return oPresent(pThis, SyncInterval, Flags);
}

static HRESULT CreateSwapChain1(IDXGIFactory* pThis,
    IUnknown* pDevice,
    DXGI_SWAP_CHAIN_DESC* pDesc,
    IDXGISwapChain** ppSwapChain
) {
    
    HRESULT result = oCreateSwapChain1(pThis, pDevice, pDesc, ppSwapChain);
    if (count < 1) {
        count++;
        return result;;
    }

    DXGI_SWAP_CHAIN_DESC swapChainDesc;
    (*ppSwapChain)->GetDesc(&swapChainDesc);

    ID3D11Device* device;
    (*ppSwapChain)->GetDevice(__uuidof(ID3D11Device), reinterpret_cast<void**>(&device));

    ID3D11DeviceContext* context;
    device->GetImmediateContext(&context);

    void** vtable = *(void***)(*ppSwapChain);

    oPresent = (tPresent)vtable[8];

    DWORD old_protection;
    VirtualProtect(&vtable[8], 8, PAGE_READWRITE, &old_protection);
    vtable[8] = (void*)hkPresent;
    VirtualProtect(&vtable[8], 8, old_protection, &old_protection);

    GUI::Init(device, *ppSwapChain, context, swapChainDesc.OutputWindow);

    return result;
}
{% endhighlight %}

![Image](/assets/TombRaider/images/box.png)

# SOTTR muscle intensity modification
I wanted to try modifing the muscle intensity in photo mode beyond the limit that the slider allows. The muscle intensity normally ranges between 0.0f and 1.0f, where 0.0f is the highest muscle intensity and 1.0f is the lowest muscle intensity. The value is modified in SOTTR.exe+0x113b17. Setting the value to a negative value does not do anything, however, you can shrink Lara's muscles by setting it to a value higher than 1.0f.

![Image](/assets/TombRaider/images/Lara_0f_muscle.jpg)
*Lara with maximum muscle intensity (0.0f)*
![Image](/assets/TombRaider/images/Lara_1f_muscle.jpg)
*Lara with minimum muscle intensity (1.0f)*
![Image](/assets/TombRaider/images/Lara_9999f_muscle.jpg)
*Lara with a cheated muscle intensity higher than 1.0f*

