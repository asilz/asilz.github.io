---
layout: post
title:  "Telltale Game Engine Animation"
date:   2024-10-4 00:36:43 +0200
categories: jekyll update
---

# Reverse engineering process
I start by running [ghidra](https://ghidra-sre.org/). Inside the symbol tree we can look at the member functions for `Animation`. We are looking for the `MetaOperation_SerializeAsync` function. To help the ghidra decompiler, we shall cast the first parameter to `Animation *` and the fourth parameter to `MetaStream *`. Now take a look at the decompiled output. If you see a lot of function pointers `Code **`, it means you are most likely missing the virtual function tables and will need to create those to be able to see which functions are called in the decompiled output. 

The `MetaOperation_SerializeAsync` functions describe how files are serialized. It will give us insight into the structure of the file. At the start of the function, we can see a call to the generic `Meta::MetaOperation_SerializeAsync`. This will iterate through all the members of the animation described in its MetaClassDescription. Let's investigate the `Animation::GetMetaClassDescription` function to figure out what is serialized. 

I was initially very intimidated looking the decompiled output, but all you need to do is to look for other calls to `GetMetaClassDescription` (Here it can be helpful to change the settings in ghidra to show templates). I can see a `long`, `Flags`, `Symbol`, `float`, `float` and a `ToolProps`. `long`, 'float' and `Flags` are 4 byte values. `Symbol` is a crc64-ecma 182 hash (8 bytes). `ToolProps` is 1 byte. 

We now return to `MetaOperation_SerializeAsync` for `Animation`. It is time to analyse the function to attempt to understand the structure of the animation file. You can rename variables, add comments and retype variables to aid you. I would also recommend having an animation file open while analyzing to confirm whether you are interpreting the decompiled output correctly. 

After some work I managed to understand how animation is read. It might be helpful to use [TelltaleDialogEditor](https://github.com/asilz/TelltaleDialogEditor) to visualize the structure as you read this. I will not describe the structure in detail here. After the DataTypes in the MetaClassDescription, we have a `long` which is the `interfaceCount` which is followed by another `long dataBufferSize`. Then we read an array `animationValueArray`. Each element in the array contains a hash which is the hash of a DataType and it also contains the number of elements of this DataType. After we read `animationValueArray`, we read DataTypes according to it. This means `Animation` just like `PropertySet` can contain any DataType. There is more data in the end of the file which I will not describe here.

Let's read a file then. Reading sk62_clementine_walk.anm from TWD DE, I find three DataTypes serialized. Two `KeyframedValue<Transform>` and a `CompressedSkeletonKeys2`. The `KeyframedValue<Transform>` values do not seem to contain much data. Most of the data seems to be contained within `CompressedSkeletonKeys2` which the engine reads as a char buffer, so the structure is not shown. The animation data has to be contained within `CompressedSkeletonKeys2` since most of the animation file is composed of it.

In the "Data Type Manager" in ghidra we search for `CompressedSkeletonKeys2`. It does exist as a DataType. We need to try and understand how the char buffer is parsed. Let's check member functions of it. The `CompressedSkeletonPoseKeys2::ComputeValue` function seems to parse the data. The function is huge. This is going to take a lot of time. It is important here to be sure that we are investigating the correct function. We do not want to waste our time analyzing a function which does not help us, but this function does indeed contain what we need. 

After some time, I was able to understand this function. I was able to write some code to parse the `CompressedSkeletonPoseKeys2` inside the file and convert the data to [glTF](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html). The animation is very incorrect however. We must investigate how the data the `CompressedSkeletonPoseKeys2::ComputeValue` function is used after it is called. Checking for references to the function, we find nothing. This is most likely due to it being called through some sort of virtual function table. We could try hunting down where it is called, but there is an easy way of finding where the function is called. We can use a debugger

I will use [x64dbg](https://x64dbg.com/) (There is also a [ghidra extension for x64dbg](https://github.com/bootleg/ret-sync), I have not tested it though). First we run TWD DE and attach the debugger. We can set a breakpoint in `CompressedSkeletonPoseKeys2::ComputeValue` and run an animation through the character viewer until we hit the breakpoint. We can now check the call stack to find calls to our function. We can see it is being called from `SkeletonInstance::_UpdatePose` farther up the call stack.

Back to ghidra. Inside `SkeletonInstance::_UpdatePose` we can see a call `SklNodeData::Update` which is probably the function responsible for updating the location of bones depending on the data contained inside `CompressedSkeletonPoseKeys2`. The decompiler output is ugly and hard to read. In this case I decide it is a better idea to guess how the data is being passed into `SklNodeData::Update` rather than actually reading. Guessing how things work instead of reading can save you many hours of work. Inside `SklNodeData::Update` we see that the data is being used along with a `boneScaleAdjust` and a `boneRotationAdjust`. We need to what these values are and how they are being calculated from the skeleton file. We check for references and see that `boneScaleAdjust` seems to be based on the `boneLength`. Inside the InternalMetaClassDescription for `Skeleton` we see that it contains a bone length, but the actual skeleton file does not contain the bone length. Other refernces to `boneScaleAdjust` are not  helpful.

Again we are stuck. When you are in a situation like this you can always attempt to guess. I will show you the structure of `Skeleton::Entry` and with a bit of thinking you might be able to figure it out how to calculate the bone length by just looking at the members. 
{% highlight c %}
struct Skeleton__Entry
{
    uint64_t jointNameSymbol;
    uint64_t parentNameSymbol;
    int32_t parentIndex;
    uint64_t mirrorBoneNameSymbol;
    int32_t mirrorBoneIndex;
    struct Vector3 localPos;
    struct Quaternion localQuaternion;
    uint32_t transformBlock;
    struct Transform restXForm;
    struct Vector3 globalTranslationScale;
    struct Vector3 localTranslationScale;
    struct Vector3 animationTranslationScale;
    uint32_t resourceGroupMembershipBlock;
    void *resourceGroupMembership; // Map<Symbol,float,std::less<Symbol>_>
    uint32_t contraintsBlock;
    struct BoneContraints contraints;
    uint32_t flags;
};
{% endhighlight %}
You might be able to guess how to calculate the bone length. I was unable to do it by just looking the data. One thing that would help finding out how the correct the value is calculated is to know the correct value for the bone length or the `boneScaleAdjust`. Let's run x64dbg again to try and dump the 'boneScaleAdjust' from memory. 





# Animation file format description

| bits  |bit size| data                       | description                                                                               |
|-------|--------|----------------------------|-------------------------------------------------------------------------------------------|
| 0-15  |16 bits | `float time`               |time of the frame                                                                          |
| 16-27 |12 bits | `uint animationBoneIndex`  |index of the bone which the frame belongs to                                               |
| 28-29 |2 bits  | `uint quaternionAxisOrder` |order in which x,y,z,w are serialized                                                      |
| 30    |1 bit   | `bool isQuaternion`        |whether the frame represents a rotation (true) or translation (false)                      |
| 31    |1 bit   | `bool isDelta`             |whether the data is an addition upon the previous data (true) or an absolute value (false) |


