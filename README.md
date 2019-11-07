# OscCore
A **performance-oriented** OSC library for Unity

#### Why Another OSC Library ?

There are already at least 4 other OSC implementations for Unity:
- [OscJack](https://github.com/keijiro/OscJack), [ExtOSC](https://github.com/Iam1337/extOSC), [UnityOSC](https://github.com/jorgegarcia/UnityOSC), & [OscSimpl](https://assetstore.unity.com/packages/tools/input-management/osc-simpl-53710)

OscCore was created largely because all of these other libraries _allocate memory for each received message_, which will cause lots of garbage collections when used with a large amount of messages.  For more on this see [performance details](#performance-details)

## Version Compatibility

Releases are checked for compatability with the latest release of these versions, and should work with anything in between.
- **2018.4.x** (LTS)
- **2019.x** (Official release)

## Installation

Download & import the .unitypackage for your platform from the [Releases](https://github.com/stella3d/OscCore/releases) page.

Proper support for the [Unity package manager](https://docs.unity3d.com/Packages/com.unity.package-manager-ui@1.8/manual/index.html) will come once I also have packages setup for the dependencies.



## Usage

#### For incoming messages

Add a `OscReceiver` component to a GameObject somewhere. 

The `OscServer` instance attached to the component is what will actually handle incoming messages - the component is just a wrapper.

The server must have its `Update()` method ticked to handle main thread queued callbacks. `OscReceiver`handles this for you. 

##### Adding address handlers

There are several different kinds of method that can be registered with a server.

###### Single method

You can register a single callback, to be executed on the server's background thread immediately when a message is received, by calling `TryAddMethod`.

If you have no need to queue a method to be called on the main thread, you probably want this one.
```csharp
class SingleCallbackExample
{
  OscServer Server { get; set; }      // get the server instance from the OscReceiver component

  void ReadValues(OscMessageValues values)
  {
      // call ReadElement methods here to extract values
  }

  public SingleCallbackExample()
  {
    // add a single callback that reads messsage values at `/layers/1/opacity`
    Server.TryAddMethod(ReadValues);
  }
}
```

###### Method pair

You can register a pair of methods to an address.

An `OscActionPair` consists of two methods, with the main thread one being optional.

1) Runs on background thread, immediate execution, just like single methods
2) Runs on main thread, queued on the next frame

This is useful for invoking UnityEvents on the main thread, or any other use case that needs a main thread api.
Read the message values in the first callback.

```csharp
class CallbackPairExample
{
  OscServer Server { get; }
  OscActionPair ActionPair { get; set;}

  float MessageValue;

  void ReadValues(OscMessageValues values) 
  {
    MessageValue = values.ReadFloatElement(0);
  }
  
  void MainThreadMethod() 
  {
    // do something with MessageValue on the main thread
  }

  public CallbackPairExample()
  {
    // create and add a pair of methods 
    ActionPair = new OscActionPair(ReadValues, MainThreadMethod);
    Server.TryAddMethodPair(ActionPair);
  }
}
```

###### Monitor Callbacks

IF you just want to inspect message, you can add a monitor callback to be able to inspect every invoming message.

A monitor callback is an `Action<BlobString, OscMessageValues>`, where the blob string is the address.  You can look at the [Monitor Window](https://github.com/stella3d/OscCore/blob/master/Editor/MonitorWindow.cs) code for an example.


#### For sending messages

[OscWriter](https://github.com/stella3d/OscCore/blob/master/Runtime/Scripts/OscWriter.cs) handles serialization of individual message elements.

[OscClient](https://github.com/stella3d/OscCore/blob/master/Runtime/Scripts/OscClient.cs) wraps a writer and sends whole messages.

Sending of complex messages with multiple elements hasn't been abstracted yet - take a look at the methods in `OscClient` to see how to send any message you need.

```csharp

OscClient Client = new OscClient("127.0.0.1", 7000);

// single a single float element message
Client.Send("/layers/1/opacity", 0.5f);

// send a blob
byte[] Blob = new byte[256];
Client.Send("/blobs", Blob, Blob.Length);

// send a string
Client.Send("/layers/3/name", "Textural");
```

## Protocol Support Details

All OSC 1.0 types, required and non-standard are supported.  

The notable parts missing from [the spec](http://opensoundcontrol.org/spec-1_0) for the initial release are:
- **Matching incoming Address Patterns**

  "_A received OSC Message must be disptched to every OSC method in the current OSC Address Space whose OSC Address matches the OSC Message's OSC Address Pattern"_
   
   Currently, an exact address match is required for incoming messages.
   If our address space has two methods:
   - `/layer/1/opacity`
   - `/layer/2/opacity`

  and we get a message at `/layer/?/opacity`, we _should_ invoke both messages.

  Right now, we would not invoke either message.  We would only invoke messages received at exactly one of the two addresses.
  This is the first thing that i think will be implemented after initial release - some of the other packages also lack this feature, and you can get far without it.

- **Syncing to a source of absolute time**

  "_An OSC server must have access to a representation of the correct current absolute time_". 

  I've implemented this, as a class that syncs to an external NTP server, but without solving clock sync i'm not ready to add it.

- **Respecting bundle timestamps**

  "_If the time represented by the OSC Time Tag is before or equal to the current time, the OSC Server should invoke the methods immediately... Otherwise the OSC Time Tag represents a time in the future, and the OSC server must store the OSC Bundle until the specified time and then invoke the appropriate OSC Methods._"

  This is simple enough to implement, but without a mechanism for clock synchronization, it could easily lead to errors.  If the sending application worked from a different time source than OscCore, events would happen at the wrong time.

## Performance Details

##### Strings and Addresses

Every OSC message starts with an "address", specified as an ascii string.  
It's perfectly reasonable to represent this address in C# as a standard `string`, which is how other libraries work.

However, because strings in C# are immutable & UTF16, every time we receive a message from the network, this now requires us to allocate a new `string`, and in the process expand the received ascii string's bytes (where each character is a single byte) to UTF16 (each `char` is two bytes).   

OscCore eliminates both 
- string allocation
- the need to convert ascii bytes to UTF16

This works through leveraging the [BlobHandles](https://github.com/stella3d/BlobHandles) package.  
Incoming message's addresses are matched directly against their unmanaged ascii bytes.  

This has two benefits 
- no memory is allocated when a message is received
- takes less CPU time to parse a message

##### 'Unsafe' Casting

OscCore uses unsafe code in order to achieve the fastest conversion from bytes to C# types possible.  

Let's look at the process of reading a float as an example. It works like this in safe code:
```csharp
byte[] m_SharedBuffer = new byte[1024];
byte[] m_SwapBuffer32 = new byte[4];

public float ReadFloatElement(int index)
{
    var offset = Offsets[index];
    m_SwapBuffer32[0] = m_SharedBuffer[offset + 3];
    m_SwapBuffer32[1] = m_SharedBuffer[offset + 2];
    m_SwapBuffer32[2] = m_SharedBuffer[offset + 1];
    m_SwapBuffer32[3] = m_SharedBuffer[offset];
    return BitConverter.ToSingle(m_SwapBuffer32, 0);
}
```

but in OscCore, we use pointer indexing to both the socket read buffer and the 4-byte swap array we use to convert big-endian floats to little-endian.  Using pointer indexing means no array bounds checking, guaranteed.

We also create these pointers on startup so it's not necessary to add extra instructions with a `fixed` statement each time.

```csharp
byte[] m_SharedBuffer = new byte[1024];
byte* SharedBufferPtr;                    
byte[] m_SwapBuffer32 = new byte[4];
byte* SwapBuffer32Ptr;

public float ReadFloatElement(int index)
{
    var offset = Offsets[index];
    SwapBuffer32Ptr[0] = SharedBufferPtr[offset + 3];
    SwapBuffer32Ptr[1] = SharedBufferPtr[offset + 2];
    SwapBuffer32Ptr[2] = SharedBufferPtr[offset + 1];
    SwapBuffer32Ptr[3] = SharedBufferPtr[offset];
    // interpret the 4 bytes in the swap buffer as a 32-bit float
    return *SwapBuffer32Ptr;
}
```

This method of reading is just as reliable and tested to be more than 3 times faster under Mono. 


