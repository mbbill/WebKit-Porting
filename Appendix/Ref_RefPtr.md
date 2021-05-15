Ref
===
Ref counted, means increase/decrease of counter when Ref<Type> gets copied or destroyed.
Usually we do `Ref<Type> v = adoptRef(*new Type())`

RefPtr
======
As `Ref`, except that it contains a pointer, thus it can be zero.
Note that, `adoptPtr` is deprecated.
Usually, `RefPtr<Type> pv = adoptRef(new Type())`

UniqueRef
=========
Not ref-counted.
```
PageConfiguration::PageConfiguration(UniqueRef<EditorClient>&& editorClient, Ref<SocketProvider>&& socketProvider, UniqueRef<LibWebRTCProvider>&& libWebRTCProvider)
    : editorClient(WTFMove(editorClient))
    , socketProvider(WTFMove(socketProvider))
    , libWebRTCProvider(WTFMove(libWebRTCProvider))

 PageConfiguration configuration(
        makeUniqueRef<WebEditorClient>(this),
        SocketProvider::create(),
        makeUniqueRef<LibWebRTCProvider>()
);
```

unique_ptr
==========
Not ref-counted, will be destroyed with owner.
Usually, `std::unique_ptr<Type> up = std::make_unique<Type>(*new Type())`

shared_ptr
==========
Not used in WebKit.

Simple Rules
============
- For RefCounted classes like `class A : public RefCounted<A>`, use Ref or RefPtr.
- For non-RefCounted classes, use unique_ptr for data members.



Guidelines From WebKit.org
==========================
We’ve developed these guidelines for use of RefPtr and Ref in WebKit code.

Local Variables
---------------
If ownership and lifetime are guaranteed, a local variable can be a raw reference or pointer.
If the code needs to hold ownership or guarantee lifetime, a local variable should be a Ref, or if it can be null, a RefPtr.

Data Members
------------
If ownership and lifetime are guaranteed, a data member can be a raw reference or pointer.
If the class needs to hold ownership or guarantee lifetime, the data member should be a Ref or RefPtr.

Function Arguments
------------------
If a function does not take ownership of an object, the argument should be a raw reference or raw pointer.
If a function does take ownership of an object, the argument should be a Ref&& or a RefPtr&&. This includes many setter functions.

Function Results
----------------
If a function’s result is an object, but ownership is not being transferred, the result should be a raw reference or raw pointer. This includes most getter functions.
If a function’s result is a new object or ownership is being transferred for any other reason, the result should be a Ref or RefPtr.

New Objects
-----------
New objects should be put into a Ref as soon as possible after creation to allow the smart pointers to do all reference counting automatically.
For RefCounted objects, the above should be done with the adoptRef function.

**Best idiom is to use a private constructor and a public create function that returns a Ref.**