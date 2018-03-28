<!--
 | Copyright (c) 2016-2018  Moddable Tech, Inc.
 |
 |   This file is part of the Moddable SDK Runtime.
 | 
 |   The Moddable SDK Runtime is free software: you can redistribute it and/or modify
 |   it under the terms of the GNU Lesser General Public License as published by
 |   the Free Software Foundation, either version 3 of the License, or
 |   (at your option) any later version.
 | 
 |   The Moddable SDK Runtime is distributed in the hope that it will be useful,
 |   but WITHOUT ANY WARRANTY; without even the implied warranty of
 |   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 |   GNU Lesser General Public License for more details.
 | 
 |   You should have received a copy of the GNU Lesser General Public License
 |   along with the Moddable SDK Runtime.  If not, see <http://www.gnu.org/licenses/>.
 |
 | This file incorporates work covered by the following copyright and  
 | permission notice:  
 |
 |       Copyright (C) 2010-2016 Marvell International Ltd.
 |       Copyright (C) 2002-2010 Kinoma, Inc.
 |
 |       Licensed under the Apache License, Version 2.0 (the "License");
 |       you may not use this file except in compliance with the License.
 |       You may obtain a copy of the License at
 |
 |        http://www.apache.org/licenses/LICENSE-2.0
 |
 |       Unless required by applicable law or agreed to in writing, software
 |       distributed under the License is distributed on an "AS IS" BASIS,
 |       WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 |       See the License for the specific language governing permissions and
 |       limitations under the License.
-->

<!-- Version: 151214a-CR

This document describes XS in C, the C interface to `xslib`, the runtime library component of the XS toolkit.
-->

# XS in C

Revised: March 27, 2018  
**See end of document for copyright and license**

## About This Document

This document describes XS in C, the C interface to the runtime of the XS JavaScript engine. Information on building XS for target software/hardware platforms is provided in the companion document [XS Platforms.md](./XS%20%Platforms.md). 

In accordance with the ECMAScript specifications, the XS runtime implements only generic features that all scripts can use. An application defines the specific features that its own scripts can use through C callbacks. An application that uses the XS runtime is a host in ECMAScript terminology. 

This document is organized into the following sections:

* [Slots](#slots) describes how to handle ECMAScript constructs in C callbacks, with examples that show the correspondences between ECMAScript and XS in C. 
 
* [Machine](#machine) introduces the main structure of the XS runtime (its virtual machine) and explains how to use the runtime to build a host and to make C callbacks available to scripts. This section concludes with an example that demonstrates how to use XS in C to implement a JavaScript class with C functions.

* A [Glossary](#glossary) includes all the terms defined or referenced in this document.

XS in C provides macros to access properties of objects. XS provides two functionally equivalent variations of many of the macros. The macros prefixed with `xs` alone are somewhat more convenient to work with but generate a larger binary code where as the macros prefixed with `xsmc` generate smaller binary code at the expense of being more difficult to use. Including the xsmc.h header file makes the `xs` versions of some operations available.

<a id="slots"></a>
##Slots

In the XS runtime, the *slot* is a fundamental storage unit. A slot is an opaque structure that is manipulated only through XS in C.

<div class="ccode"></div>

	typedef struct xsSlotRecord xsSlot
	struct xsSlotRecord {
		void* data[4];
	};

###Slot types
There are nine types of slots:
 
<div class="ccode"></div>

	enum {
		xsUndefinedType,
		xsNullType,
		xsBooleanType,
		xsIntegerType,
		xsNumberType,
		xsStringType,
		xsStringXType,
		xsSymbolType,
		xsReferenceType 
	}
	typedef char xsType;
 
The undefined, null, boolean, number, string, and symbol slots correspond to the ECMAScript primitive types. The reference slot corresponds to the ECMAScript `reference` type. The integer and stringx slots are optimizations that are unobservable by scripts. The integer slot is equivalent to the number slot, but allowing floating point operations to be bypassed. The stringx slot is equivalent to the string slot, but uses a string in place (e.g. in ROM) without making copy.
 
#####In ECMAScript:

	undefined
	null
	false
	true
	0
	0.0
	"foo"

#####In C:

<div class="ccode"></div>

	xsUndefined;
	xsNull;
	xsFalse;
	xsTrue;
	xsInteger(0);
	xsNumber(0.0);
	xsString("foo")

The `xsTypeOf` macro returns the type of a slot. It is similar to the ECMAScript `typeof` keyword.

```javascript
xsType xsTypeOf(xsSlot theSlot)
xsType xsmcTypeOf(xsSlot theSlot)
```

| | 
| --- | 
| `theSlot`| 

> The slot to test

| |
| --- |
| Returns | 

> The type of the slot

#####In ECMAScript:

	switch(typeof arguments[0]) {
		case "undefined": break;
		/* Null is an object. */
		case "boolean": break;	
		/* Integers are numbers */	
		case "number": break;
		/* StringX is a string */
		case "string": break;
		case "symbol": break;
		case "object": break;
		case "function": break;
	}	

#####In C:

<div class="ccode"></div>

	switch(xsTypeOf(xsArg(0))) {
		case xsUndefinedType: break;
		case xsNullType: break;
		case xsBooleanType: break;
		case xsIntegerType: break;
		case xsNumberType: break;
		case xsStringType: break;
		case xsSymbolType: break;
		case xsReferenceType: break;  /* Objects and functions are references. */
	}		


###Primitives

The undefined, null, boolean, integer, number, string, and symbol slots (collectively known as *direct slots*) correspond to the ECMAScript primitive types, with the integer and stringx slots added as optimizations.

The undefined and null slots contain no value. The `xsUndefined` and `xsNull` macros return slots of those types.

`xsSlot xsUndefined`

| |
| --- |
| Returns| 

> An undefined slot

Use the `xsmcSetUndefined` macro to set a slot value to `undefined`.

```javascript
void xsmcSetUndefined(xsSlot theSlot)
```

| |
| --- |
| `theSlot` | 

> The slot to set to `undefined`

`xsSlot xsNull`

| |
| --- |
| Returns| 

> A null slot

Use the `xsmcSetNull` macro to set a slot value to `null`.

```javascript
void xsmcSetNull(xsSlot theSlot)
```

| |
| --- |
| `theSlot` | 

> The slot to set to `null`

The remaining direct slots contain values of the corresponding type.

<div class="ccode"></div>

	typedef char xsBooleanValue;  
	typedef long xsIntegerValue;  
	typedef double xsNumberValue;  
	typedef char* xsStringValue;

The following macros return slots of each of these types (set to a particular value) or access/set the value in a slot. When accessing the value in a slot, you specify a desired type; the slot is coerced to the requested type if necessary, and the value is returned. The string macros listed here are discussed further at the end of the list.

`xsSlot xsTrue`

| |
| --- |
| Returns| 

> A boolean slot containing `true`

`xsSlot xsFalse`

| |
| --- |
| Returns| 

> A boolean slot containing `false`

`xsSlot xsBoolean(xsBooleanValue theValue)`

| | 
| --- | 
| `theValue`| 

> The value to be contained in the slot

| |
| --- |
| Returns| 

> A boolean slot

```javascript
xsBooleanValue xsToBoolean(xsSlot theSlot)
xsBooleanValue xsmcToBoolean(xsSlot theSlot)
```

| | 
| --- | 
| `theSlot`| 

> The slot to coerce to boolean

| |
| --- |
| Returns| 

> The value contained in the slot

Use the `xsmcSetFalse` macro to set a slot value to `false`.

```javascript
void xsmcSetFalse(xsSlot theSlot)
```

| |
| --- |
| `theSlot` | 

> The slot to set false

Use the `xsmcSetTrue` macro to set a slot value to `true`.

```javascript
void xsmcSetTrue(xsSlot theSlot)
```

| |
| --- |
| `theSlot` | 

> The slot to set true

Use the `xsmcSetBoolean` macro to set a slot value to `true` or `false`.

```javascript
void xsmcSetBoolean(xsSlot theSlot, xsBooleanValue theValue)
```

| |
| --- |
| `theSlot` | 

> The slot to set

| |
| --- |
| `theValue` | 

> The boolean `true` or `false` value to set

`xsSlot xsInteger(xsIntegerValue theValue)`

| | 
| --- | 
| `theValue`| 

> The value to be contained in the slot

| |
| --- |
| Returns| 

> An integer slot

```javascript
xsIntegerValue xsToInteger(xsSlot theSlot)
xsIntegerValue xsmcToInteger(xsSlot theSlot)
```

| | 
| --- | 
| `theSlot`| 

> The slot to coerce to integer

| |
| --- |
| Returns| 

> The value contained in the slot

Use the `xsmcSetInteger` macro to set a slot value to an integer.

```javascript
void xsmcSetInteger(xsSlot theSlot, xsIntegerValue theValue)
```

| |
| --- |
| `theSlot` | 

> The slot to set

| |
| --- |
| `theValue` | 

> The integer value to set
	
`xsSlot xsNumber(xsNumberValue theValue)`

| | 
| --- | 
| `theValue`| 

> The value to be contained in the slot

| |
| --- |
| Returns| 

> A number slot

```javascript
xsNumberValue xsToNumber(xsSlot theSlot)
xsNumberValue xsmcToNumber(xsSlot theSlot)
```

| | 
| --- | 
| `theSlot`| 

> The slot to coerce to number

| |
| --- |
| Returns| 

> The value contained in the slot

Use the `xsmcSetNumber` macro to set a slot value to a number.

```javascript
void xsmcSetNumber(xsSlot theSlot, xsNumberValue theValue)
```

| |
| --- |
| `theSlot` | 

> The slot to set

| |
| --- |
| `theValue` | 

> The number value to set

`xsSlot xsString(xsStringValue theValue)`

| | 
| --- | 
| `theValue`| 

> The value to be contained in the slot

| |
| --- |
| Returns| 

> A string slot

```javascript
xsStringValue xsToString(xsSlot theSlot)
xsStringValue xsmcToString(xsSlot theSlot)
```

| | 
| --- | 
| `theSlot`| 

> The slot to coerce to string

| |
| --- |
| Returns| 

> The string contained in the slot

Use the `xsmcSetString` macro to set a slot value to a string.

```javascript
void xsmcSetString(xsSlot theSlot, xsStringValue theValue)
```

| |
| --- |
| `theSlot` | 

> The slot to set

| |
| --- |
| `theValue` | 

> The string value to set

```javascript
xsStringValue xsToStringBuffer(xsSlot theSlot, xsStringValue theBuffer, xsIntegerValue theSize)
xsStringValue xsmcToStringBuffer(xsSlot theSlot, xsStringValue theBuffer, xsIntegerValue theSize)
```

| | 
| --- | 
| `theSlot`| 

> The slot to coerce to string

| | 
| --- | 
| `theBuffer`| 

> A buffer to copy the string into

| | 
| --- | 
| `theSize`| 

> The size of the buffer

| |
| --- |
| Returns| 

> The buffer containing the copy of the string

A string value is a pointer to a UTF-8 C string. The XS runtime virtual machine and the garbage collector manage UTF-8 C strings used by scripts. C constants, C globals, or C locals can safely be passed to the `xsString` macro, since it duplicates its parameter. 

For speed, the `xsToString` macro returns the value contained in the slot itself, a pointer to the string in the memory managed by XS. Since the XS runtime can compact memory containing string values, the result of the `xsToString` macro cannot be used across or in other macros of XS in C. The ECMAScript language specification forbids modifying the string in place.

To copy the string value, use the `xsToStringBuffer` or `xsmcToStringBuffer` macro. The buffer provided has to be large enough to hold a copy of the string value. 

Use the `xsStringBuffer` or `xsmcSetStringBuffer` macro to set a slot value to a string buffer. The string is copied into an allocated buffer.

```javascript
xsSlot xsStringBuffer(void *theData, xsIntegerValue theSize)
```

| | 
| --- | 
| `theData`| 

> A pointer to the data to copy into the string buffer, or `NULL` to leave the string buffer data uninitialized
 
| | 
| --- | 
| `theSize`| 

> The data size to copy in bytes

| | 
| --- | 
| Returns | 

> A reference to the new string buffer instance 

```javascript
void xsmcSetStringBuffer(xsSlot theSlot, xsStringValue theValue, xsIntegerValue theSize)
```

| |
| --- |
| `theSlot` | 

> The slot to set

| |
| --- |
| `theValue` | 

> The string value to set

| |
| --- |
| `theSize` | 

> The size of the string in bytes

###ArrayBuffer

In ECMAScript an `ArrayBuffer` is commonly used to store fixed length binary data. The following XS in C macros are provided to create `ArrayBuffer` instances and access their properties and data.

The `xsArrayBuffer` macro creates an `ArrayBuffer` instance.

`xsSlot xsArrayBuffer(void *theData, xsIntegerValue theSize)`
 
| | 
| --- | 
| `theData`| 

> A pointer to the data to copy into the `ArrayBuffer`, or `NULL` to leave the `ArrayBuffer` data uninitialized

| | 
| --- | 
| `theSize`| 

> The size of the data in bytes

| | 
| --- | 
| Returns | 

> A reference to the new `ArrayBuffer` instance 

Use the `xsGetArrayBufferData` macro to copy bytes from the `ArrayBuffer`.

`void xsGetArrayBufferData(xsSlot theSlot, xsIntegerValue theOffset, void *theData, xsIntegerValue theSize)`
`void xsmcGetArrayBufferData(xsSlot theSlot, xsIntegerValue theOffset, void *theData, xsIntegerValue theSize)`
 
| | 
| --- | 
| `theSlot`| 

> The `ArrayBuffer` slot

| | 
| --- | 
| `theOffset`| 

> The starting byte offset to get the data

| | 
| --- | 
| `theData`| 

> The data pointer to get the `ArrayBuffer` data
 
| | 
| --- | 
| `theSize`| 

> The data size to copy in bytes

Use the `xsSetArrayBufferData` macro to copy bytes into the `ArrayBuffer`.

`void xsSetArrayBufferData(xsSlot theSlot, xsIntegerValue theOffset, void *theData, xsIntegerValue theSize)`
`void xsmcSetArrayBufferData(xsSlot theSlot, xsIntegerValue theOffset, void *theData, xsIntegerValue theSize)`
 
| | 
| --- | 
| `theSlot`| 

> The `ArrayBuffer` slot

| | 
| --- | 
| `theOffset`| 

> The starting byte offset to set the data

| | 
| --- | 
| `theData`| 

> The data pointer to set the `ArrayBuffer` data
 
| | 
| --- | 
| `theSize`| 

> The data size to copy in bytes


The `xsmcSetArrayBuffer` macro creates an `ArrayBuffer` instance initialized from the provided data.

`void xsmcSetArrayBuffer(xsSlot theSlot, void *theData, xsIntegerValue theSize)`

| | 
| --- | 
| `theSlot`| 

> The `ArrayBuffer` slot

| | 
| --- | 
| `theData`| 

> The data pointer to set the `ArrayBuffer` data
 
| | 
| --- | 
| `theSize`| 

> The data size to copy in bytes

Use the `xsGetArrayBufferLength` macro to get the `ArrayBuffer` length.

`xsIntegerValue xsGetArrayBufferLength(xsSlot theSlot)`
 
| | 
| --- | 
| `theSlot`| 

> The `ArrayBuffer` slot

| | 
| --- | 
| Returns | 

> The size of the `ArrayBuffer` in bytes

Use the `xsSetArrayBufferLength` macro to set the `ArrayBuffer` length.

`void xsSetArrayBufferLength(xsSlot theSlot, xsIntegerValue theSize)`
 
| | 
| --- | 
| `theSlot`| 

> The `ArrayBuffer` slot

| | 
| --- | 
| theSize | 

> The size of the `ArrayBuffer` data in bytes. If the size of the buffer is increased, the new data is initialized to 0.

To get a pointer to the data of an `ArrayBuffer`, use the `xsToArrayBuffer` macro.

`void *xsToArrayBuffer(xsSlot theSlot)`
`void *xsmcToArrayBuffer(xsSlot theSlot)`
 
| | 
| --- | 
| `theSlot`| 

> The `ArrayBuffer` slot

| | 
| --- | 
| Returns | 

> A pointer to the `ArrayBuffer` data

For speed, the `xsToArrayBuffer ` macro returns the value contained in the slot itself, a pointer to the buffer in the memory managed by XS. Since the XS runtime can compact memory containing string values, the result of the `xsToArrayBuffer ` macro cannot be used across or in other macros of XS in C. 

###Instances and Prototypes

In XS in C, as in ECMAScript, an object can inherit properties from another object, which can inherit from another object, and so on; the inheriting object is the *instance*, and the object from which it inherits is the *prototype*.

Reference slots (type `xsReferenceType`) are *indirect* slots: they contain a reference to an instance of an object, function, array, and so on. Instances themselves are made of slots that are the properties of the instance (or, for an array, the items of the instance).

XS in C defines the following macros to refer to the prototypes created by the XS runtime:

	xsSlot xsObjectPrototype
	xsSlot xsFunctionPrototype
	xsSlot xsArrayPrototype
	xsSlot xsStringPrototype
	xsSlot xsBooleanPrototype
	xsSlot xsNumberPrototype
	xsSlot xsDatePrototype
	xsSlot xsRegExpPrototype
	xsSlot xsHostPrototype
	xsSlot xsErrorPrototype
	xsSlot xsEvalErrorPrototype
	xsSlot xsRangeErrorPrototype
	xsSlot xsReferenceErrorPrototype
	xsSlot xsSyntaxErrorPrototype
	xsSlot xsTypeErrorPrototype
	xsSlot xsURIErrorPrototype
	xsSlot xsSymbolPrototype
	xsSlot xsArrayBufferPrototype
	xsSlot xsDataViewPrototype
	xsSlot xsTypedArrayPrototype
	xsSlot xsMapPrototype
	xsSlot xsSetPrototype
	xsSlot xsWeakMapPrototype
	xsSlot xsWeakSetPrototype
	xsSlot xsPromisePrototype
	xsSlot xsProxyPrototype

| |
| --- |
| Returns | 

> A reference to the prototype instance.

For convenience, the `xsNewArray` and `xsNewObject` macros are provided to create array and object instances.

`xsSlot xsNewArray(xsIntegerValue theLength)`
`xsSlot xsmcNewArray(xsIntegerValue theLength)`

| | 
| --- | 
| `theLength`| 

> The array length property to set

| | 
| --- | 
| Returns | 

> A reference to the new array instance 

#####In ECMAScript:

	new Array(5);

#####In C:

<div class="ccode"></div>

	xsNewArray(5);

To create a new object instance, use the`xsNewObject` macro.

```javascript
xsSlot xsNewObject()
xsSlot xsmcNewObject()
```

| | 
| --- | 
| Returns | 

> A reference to the new object instance 

#####In ECMAScript:

	new Object();

#####In C:

<div class="ccode"></div>

	xsNewObject();

The `xsmcSetNewObject` macro is functionally equivalent to the `xsNewObject` macro. The property is returned in the slot provided.

`void xsmcSetNewObject(theSlot theSlot)`

| |
| --- |
| `theSlot` | 

> The result slot 

#####In ECMAScript:

	new Object();

#####In C:

<div class="ccode"></div>

	xsVars(1);
	xsmcNewObject(xsVar(0));

To test whether an instance has a particular prototype, directly or indirectly (that is, one or more levels up in the prototype hierarchy), use the `xsIsInstanceOf` macro.

```javascript
xsBooleanValue xsIsInstanceOf(xsSlot theInstance, xsSlot thePrototype)
xsBooleanValue xsmcIsInstanceOf(xsSlot theInstance, xsSlot thePrototype)
```

| | 
| --- | 
| `theInstance`| 

> A reference to the instance to test

| | 
| --- | 
| `thePrototype`| 

> A reference to the prototype to test

| | 
| --- | 
| Returns | 

> `true` if the instance has the prototype, `false` otherwise

The `xsIsInstanceOf` macro has no equivalent in ECMAScript; scripts test instances through *constructors* rather than directly through prototypes. A constructor is a function that has a `prototype` property that is used to test instances with `isPrototypeOf`.

#####In ECMAScript:

	if (Object.prototype.isPrototypeOf(this))
		return new Object();

#####In C:

<div class="ccode"></div>

	if (xsIsInstanceOf(xsThis, xsObjectPrototype))
		xsResult = xsNewObject();

###Identifiers

In ECMAScript, the properties of an object are identified by strings, numbers, and symbols. Similarly in XS in C you can access properties with strings, numbers, and symbols through the xs*At macros described below. The `xsIndex` type is used to identify properties by index.

<div class="ccode"></div>

	typedef short xsIndex;

The properties of an object are identified by negative indexes, and the items of an array are identified by positive indexes.

<a id="xsid"></a>
####xsID

XS in C defines the `xsID` macro to convert a string value corresponding to an ECMAScript property name into an identifier. 

`xsIndex xsID(xsStringValue theValue)`

| | 
| --- | 
| `theValue` | 

> The string to convert

| |
| --- |
| Returns | 

> The identifier (a negative index)

For a given virtual machine, the same string value is always converted to the same identifier, allowing frequently used identifiers to be cached.

For performance, XS in C also supports accessing properties by identifiers generated by the XS compiler, e.g. `xsID_property`. The type of the `xsID_*` property is `xsIndex`. These identifiers provide an optimization that can be used by the xsGet* and xsSet* macros described below. the `xsID_*` properties are functionally equivalent to using the `xsID()` macro and used in all the examples.

In the C examples below, the `xsGet` macro (discussed in the next section) takes as its second argument the identifier of the property or item to get.

#####In ECMAScript:

	this.foo
	this[0]

#####In C:

<div class="ccode"></div>

	xsGet(xsThis, xsID_foo);
	xsGet(xsThis, 0);

####xsIsID

Use the `xsIsID` macro to test whether a given string corresponds to an existing property name.

`xsBooleanValue xsIsID(xsStringValue theValue)`

| | 
| --- | 
| `theValue` | 

> The string to test

| |
| --- |
| Returns | 

> `true` if the string is an identifier, `false` otherwise


### Properties 

This section describes the macros related to accessing properties of objects (or items of arrays), as summarized in Table 1.

**Table 1.** Property-Related Macros

<table class="normalTable">
  <tbody>
    <tr>
      <th scope="col">Macro</th>
      <th scope="col">Description</th>
    </tr>
    <tr>
      <td><code>xsGlobal</code></td>
      <td>Returns a special instance made of global properties available to scripts</td>
    </tr> 
    <tr>
      <td><code>xsDefine</code></td>
      <td>Defines a new property or item of an instance with attributes</td>
    </tr> 
      <td><code>xsDefineAt</code></td>
      <td>Defines a new property or item of an instance by slot with attributes</td>
    </tr> 
    <tr>
      <td><code>xsHas, xsmcHas</code></td>
      <td>Tests whether an instance has a particular property</td>
    </tr> 
    <tr>
      <td><code>xsHasAt</code></td>
      <td>Tests whether an instance has a particular property by key</td>
    </tr> 
    <tr>
      <td><code>xsGet, xsmcGet</code></td>
      <td>Gets a property or item of an instances</td>
    </tr> 
    <tr>
      <td><code>xsGetAt, xsmcGetAt</code></td>
      <td>Gets a property or item of an array instance by key</td>
    </tr> 
	<tr>
      <td><code>xsSet, xsmcSet</code></td>
      <td>Sets a property or item of an instance</td>
    </tr> 
	<tr>
      <td><code>xsSetAt, xsmcSetAt</code></td>
      <td>Sets a property or item of an array instance at a specified index</td>
	</tr> 
    <tr>
      <td><code>xsDelete, xsmcDelete</code></td>
      <td>Deletes a property or item of an instance</td>
    </tr> 
    <tr>
      <td><code>xsDeleteAt, xsmcDeleteAt</code></td>
      <td>Deletes a property or item of an instance by key</td>
    </tr> 
    <tr>
      <td><code>xsCall0</code> ... <code>xsCall7, xsmcCall</code></td>
      <td>Calls the function referred to by a property or item of an instance</td>
    </tr> 
    <tr>
      <td><code>xsNew0</code> ... <code>xsNew7, xsmcNew</code></td>
      <td>Calls the constructor referred to by a property or item of an instance</td>
    </tr>     
    <tr>
      <td><code>xsTest, xsmcTest</code></td>
      <td>Takes a value of any type and determines whether it is true or false</td>
    </tr>     
    <tr>
      <td><code>xsEnumerate</code></td>
      <td>Enumerates the properties of an instance</td>
    </tr>     
  </tbody>
</table>


> Some of the examples below use variable slots reserved on the stack with the `xsVars` macro; See [`Arguments and Variables`](#xsvars).

#### xsGlobal

Globals available to scripts are properties of a special instance referred to using the `xsGlobal` macro in XS in C.

`xsSlot xsGlobal`

| |
| --- |
| Returns | 

> A reference to the special instance made of globals

You can use the `xsGet`, `xsSet`, `xsDelete`, `xsCall*`, and `xsNew*` macros with the `xsGlobal` macro as the first parameter. Examples are shown in the sections describing those macros.

#### xsDefine

To define a new property or item of an instance with attributes, use the `xsDefine` macro. The attributes of the property are set using one or more of the following constants. 

<div class="ccode"></div>

	enum {
		xsDefault = 0,
		xsDontDelete = 2,
		xsDontEnum = 4,
		xsDontSet = 8,
		xsStatic = 16,
		xsIsGetter = 32,
		xsIsSetter = 64,
		xsChangeAll = 30
	} 
	typedef unsigned char xsAttribute;


`void xsDefine(xsSlot theThis, xsIndex theIndex, xsSlot theParam, xsAttribute theAttributes)`

| |
| --- |
| `theThis` | 

> A reference to the instance that will have the property or item

| |
| --- |
| `theIndex` | 

> The identifier of the property or item to define

| |
| --- |
| `theParam` | 

> The value of the property or item to define

| |
| --- |
| `theAttributes` | 

> A combination of attributes to set.

For `theAttributes`, specify the constants corresponding to the attributes you want to set (the others being cleared).

The `xsDontDelete`, `xsDontEnum`, and `xsDontSet` attributes correspond to the ECMAScript `DontDelete`, `DontEnum`, and `ReadOnly` attributes. By default a property can be deleted, enumerated, and set.

When a property is created, if the prototype of the instance has a property with the same name, its attributes are inherited; otherwise, by default, a property can be deleted, enumerated, and set, and can be used by scripts.

#####In ECMAScript:
	
	Object.defineProperty(this, "foo", 7, { writable: true, enumerable: true, configurable: true });

#####In C:

<div class="ccode"></div>

	xsDefine(xsThis, xsID_foo, xsInteger(7), xsDefault);

#### xsDefineAt

To define a new property or item of an instance by slot with attributes, use the `xsDefineAt` macro. The `xsDefineAt` macro is functionally equivalent to the `xsDefine` macro, except that a slot is used to identify the property or item to define.

<div class="ccode"></div>

`void xsDefineAt(xsSlot theThis, xsSlot theSlot, xsSlot theParam, xsAttribute theAttributes)`

| |
| --- |
| `theThis` | 

> A reference to the instance that will have the property or item

| |
| --- |
| `theSlot` | 

> The slot of the property or item to define

| |
| --- |
| `theParam` | 

> The value of the property or item to define

| |
| --- |
| `theAttributes` | 

> A combination of attributes to set.

#####In ECMAScript:
	
	Object.defineProperty(this, "foo", 7, { writable: true, enumerable: true, configurable: true });
	Object.defineProperty(this, 5, 7, { writable: true, enumerable: true, configurable: true });

#####In C:

<div class="ccode"></div>

	xsDefineAt(xsThis, xsString("foo"), xsInteger(7), xsDefault);
	xsDefineAt(xsThis, xsInteger(5), xsInteger(7), xsDefault);

#### xsHas

To test whether an instance has a property corresponding to a particular ECMAScript property name, use the `xsHas` macro. This macro is similar to the ECMAScript `in` keyword.

```javascript
xsBooleanValue xsHas(xsSlot theThis, xsIndex theIndex)
xsBooleanValue xsmcHas(xsSlot theThis, xsIndex theIndex)
```

| |
| --- |
| `theThis` | 

> A reference to the instance to test


| |
| --- |
| `theIndex` | 

> The identifier of the property to test


| |
| --- |
| Returns | 

> `true` if the instance has the property, `false` otherwise


#####In ECMAScript:

	if ("foo" in this)

#####In C:

<div class="ccode"></div>

	if (xsHas(xsThis, xsID("foo")));


#### xsHasAt

To test whether an instance has a property corresponding to a particular ECMAScript property key, use the `xsHasAt` macro.

```javascript
xsBooleanValue xsmcHasAt(xsSlot theObject, xsSlot theKey)
```

| |
| --- |
| `theObject ` | 

> A reference to the instance to test


| |
| --- |
| `theKey` | 

> The key of the property to test


| |
| --- |
| Returns | 

> `true` if the instance has the property, `false` otherwise


#####In ECMAScript:

	if (7 in this)

#####In C:

<div class="ccode"></div>

	if (xsHasAt(xsThis, xsInteger(7));


#### xsGet

To get a property or item of an instance, use the `xsGet` macro. If the property or item is not defined by the instance or its prototypes, the macro returns `xsUndefined`.

`xsSlot xsGet(xsSlot theThis, xsIndex theIndex)`

| |
| --- |
| `theThis` | 

> A reference to the instance that has the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to get


| |
| --- |
| Returns | 

> A slot containing what is contained in the property or item


#####In ECMAScript:

	foo
	this.foo
	this[0]

#####In C:

<div class="ccode"></div>

	xsGet(xsGlobal, xsID("foo"));
	xsGet(xsThis, xsID("foo"));
	xsGet(xsThis, 0);

#### xsmcGet

The `xsmcGet` macro is functionally equivalent to the `xsGet` macro. The property is returned in the slot provided.

`void xsmcGet(xsSlot theSlot, xsSlot theThis, xsIndex theIndex)`

| |
| --- |
| `theSlot` | 

> The slot to contain the property or item

| |
| --- |
| `theThis` | 

> A reference to the instance that has the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to get


#####In ECMAScript:

	foo
	this.foo
	this[0]

#####In C:

<div class="ccode"></div>

	xsVars(1);
	xsmcGet(xsVar(0), xsGlobal, xsID_foo);
	xsmcGet(xsVar(0), xsThis, xsID_foo);
	xsmcGet(xsVar(0), xsThis, 0);

#### xsGetAt

To get a property or item of an array instance with a specified name, index or symbol, use the `xsGetAt` macro. If the property or item is not defined by the instance or its prototypes, the macro returns `xsUndefined`.

`xsSlot xsGetAt(xsSlot theObject, xsSlot theKey)`

| |
| --- |
| `theObject ` | 

> A reference to the object that has the property or item


| |
| --- |
| `theKey` | 

> The key of the property or item to get


| |
| --- |
| Returns | 

> A slot containing what is contained in the property or item


#####In ECMAScript:

	this.foo[3]

#####In C:

<div class="ccode"></div>

	xsVars(2);
	xsVar(0) = xsGet(xsThis, xsID_foo);
	xsVar(1) = xsGetAt(xsVar(0), xsInteger(3));

#### xsmcGetAt

The `xsmcGetAt` macro is functionally equivalent to the `xsGetAt` macro. The property is returned in the slot provided.


`void xsmcGetAt(xsSlot theSlot, xsSlot theObject, xsSlot theKey)`

| |
| --- |
| `theSlot` | 

> The slot to contain the property or item
 
| |
| --- |
| `theObject` | 

> A reference to the object that has the property or item

| |
| --- |
| `theKey` | 

> The key of the property or item to get

#####In ECMAScript:

	var tmp, tmp2;
	tmp = this.foo
	tmp2 = tmp[3]

#####In C:

<div class="ccode"></div>

	xsVars(2);
	xsmcGet(xsVar(0), xsThis, xsID_foo);
	xsmcGetAt(xsVar(1), xsVar(0), xsInteger(3));

#### xsSet

To set a property or item of an instance, use the `xsSet` macro. If the property or item is not defined by the instance, this macro inserts it into the instance.

```javascript
void xsSet(xsSlot theThis, xsIndex theIndex, xsSlot theParam)
void xsmcSet(xsSlot theThis, xsIndex theIndex, xsSlot theParam)
```

| |
| --- |
| `theThis` | 

> A reference to the instance that will have the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to set

| |
| --- |
| `theParam` | 

> The value of the property or item to set

#####In ECMAScript:

	foo = 0
	this.foo = 1
	this[0] = 2

#####In C:

<div class="ccode"></div>

	xsSet(xsGlobal, xsID_foo, xsInteger(0));
	xsSet(xsThis, xsID_foo, xsInteger(1));
	xsSet(xsThis, 0, xsInteger(2));


#### xsSetAt

To set a property or item of an array instance by key, use the `xsSetAt` macro. If the property or item is not defined by the instance, this macro inserts it into the instance.

```javascript
void xsSetAt(xsSlot theObject, xsSlot theKey, xsSlot theValue)
void xsmcSetAt(xsSlot theObject, xsSlot theKey, xsSlot theValue)
```

| |
| --- |
| `theObject` | 

> A reference to the object that has the property or item

| |
| --- |
| `theKey` | 

> The key of the property or item to set

| |
| --- |
| `theValue` | 

> The value of the property or item to set

#####In ECMAScript:

	this.foo[3] = 7

#####In C:

<div class="ccode"></div>

	xsVars(1);
	xsVar(0) = xsGet(xsThis, xsID_foo);
	xsSetAt(xsVar(0), xsInteger(3), xsInteger(7));

#### xsDelete

To delete a property or item of an instance, use the `xsDelete` macro. If the property or item is not defined by the instance, this macro has no effect.

```javascript
void xsDelete(xsSlot theThis, xsIndex theIndex)
void xsmcDelete(xsSlot theThis, xsIndex theIndex)
```

| |
| --- |
| `theThis` | 

> A reference to the instance that has the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to delete


#####In ECMAScript:

	delete foo
	delete this.foo
	delete this[0]

#####In C:

<div class="ccode"></div>

	xsDelete(xsGlobal, xsID_foo);
	xsDelete(xsThis, xsID_foo);
	xsDelete(xsThis, 0);


#### xsDeleteAt, xsmcDeleteAt

To delete a property or item of an instance by key, use the `xsDeleteAt` macro. If the property or item is not defined by the instance, this macro has no effect.

```javascript
void xsDeleteAt(xsSlot theObject, xsSlot theKey)
void xsmcDeleteAt(xsSlot theObject, xsSlot theKey)
```

| |
| --- |
| `theObject` | 

> A reference to the object that has the property or item


| |
| --- |
| `theKey` | 

> The key of the property or item to delete


#####In ECMAScript:

	delete this.foo
	delete this[0]

#####In C:

<div class="ccode"></div>

	xsDeleteAt(xsThis, xsID_foo);
	xsDeleteAt(xsThis, xsInteger(0));


#### xsCall*

When a property or item of an instance is a reference to a function, you can call the function with one of the `xsCall*` macros (where `*` is `0` through `7`, representing the number of parameter slots passed). If the property or item is not defined by the instance or its prototypes or is not a reference to a function, the `xsCall*` macro throws an exception.

`xsSlot xsCall0(xsSlot theThis, xsIndex theIndex)`
  
`xsSlot xsCall1(xsSlot theThis, xsIndex theIndex, xsSlot theParam0)`
 
`xsSlot xsCall2(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1)`

`xsSlot xsCall3(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2)`

`xsSlot xsCall4(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3)`

`xsSlot xsCall5(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3, xsSlot theParam4)`

`xsSlot xsCall6(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3, xsSlot theParam4, xsSlot theParam5)`

`xsSlot xsCall7(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3, xsSlot theParam4, xsSlot theParam5, xsSlot theParam6)`

| |
| --- |
| `theThis` | 

> A reference to the instance that will have the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to call


| |
| --- |
| `theParam0` ... `theParam6` | 

> The parameter slots to pass to the function

| |
| --- |
| Returns | 

> The result slot of the function


#####In ECMAScript:

	foo()
	this.foo(1)
	this[0](2, 3)

#####In C:

<div class="ccode"></div>

	xsCall0(xsGlobal, xsID_foo);
	xsCall1(xsThis, xsID_foo, xsInteger(1));
	xsCall2(xsThis, 0, xsInteger(2), xsInteger(3));

#### xsmcCall

The `xsmcCall` macro is functionally equivalent to the `xsCall*` macros. The result and parameter slots are provided as function parameters.

`void xsmcCall(xsSlot xsSlot, xsSlot theThis, xsIndex theIndex, ...)`

| |
| --- |
| `theSlot` | 

> The result slot

| |
| --- |
| `theThis` | 

> A reference to the instance that will have the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to call


| |
| --- |
| ... | 

> The variable length parameter slots to pass to the function


#####In ECMAScript:

	foo(1)
	this.foo(1)
	this[0](2, 3)

#####In C:
<div class="ccode"></div>

	xsVars(3);
	xsmcSetInteger(xsVar(0), 1);
	xsmcSetInteger(xsVar(1), 2);
	xsmcSetInteger(xsVar(2), 3);
	xsmcCall(xsResult, xsGlobal, xsID_foo, xsVar(0));
	xsmcCall(xsResult, xsThis, xsID_foo, xsVar(0));
	xsmcCall(xsResult, xsThis, 0, xsVar(1), xsVar(2));
	
<a id="xsnew"></a>
#### xsNew*

When a property or item of an instance is a reference to a constructor, you can call the constructor with one of the `xsNew*` macros (where `*` is `0` through `7`, representing the number of parameter slots passed). If the property or item is not defined by the instance or its prototypes or is not a reference to a constructor, the `xsNew*` macro throws an exception.

`xsSlot xsNew0(xsSlot theThis, xsIndex theIndex)`

`xsSlot xsNew1(xsSlot theThis, xsIndex theIndex, xsSlot theParam0)`

`xsSlot xsNew2(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1)`

`xsSlot xsNew3(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2)`

`xsSlot xsNew4(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3)`

`xsSlot xsNew5(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3, xsSlot theParam4)`

`xsSlot xsNew6(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3, xsSlot theParam4, xsSlot theParam5)`

`xsSlot xsNew7(xsSlot theThis, xsIndex theIndex, xsSlot theParam0, xsSlot theParam1, xsSlot theParam2, xsSlot theParam3, xsSlot theParam4, xsSlot theParam5, xsSlot theParam6)`

| |
| --- |
| `theThis` | 

> A reference to the instance that has the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to call

| |
| --- |
| `theParam0` ... `theParam6` | 

> The parameter slots to pass to the constructor


| |
| --- |
| Returns | 

> The result slot of the constructor


#####In ECMAScript:

	new foo()
	new this.foo(1)
	new this[0](2, 3)

#####In C:

<div class="ccode"></div>

	xsNew0(xsGlobal, xsID_foo);
	xsNew1(xsThis, xsID_foo, xsInteger(1));
	xsNew2(xsThis, 0, xsInteger(2), xsInteger(3));


#### xsmcNew

The `xsmcNew` macro is functionally equivalent to the `xsNew*` macros. The result and parameter slots are provided as function parameters.


`void xsmcNew(xsSlot theSlot, xsSlot theThis, xsIndex theIndex, ...)`

| |
| --- |
| `theSlot` | 

> The result slot of the constructor

| |
| --- |
| `theThis` | 

> A reference to the instance that has the property or item


| |
| --- |
| `theIndex` | 

> The identifier of the property or item to call

| |
| --- |
| ... | 

> The variable length parameter slots to pass to the function


#####In ECMAScript:

	new foo(1)
	new this.foo(1)
	new this[0](2, 3)

#####In C:

<div class="ccode"></div>

	xsVars(3);
	xsmcSetInteger(xsVar(0), 1);
	xsmcSetInteger(xsVar(1), 2);
	xsmcSetInteger(xsVar(2), 3);
	xsmcNew(xsResult, xsGlobal, xsID_foo, xsVar(0));
	xsmcNew(xsResult, xsThis, xsID_foo, xsVar(0));
	xsmcNew(xsResult, xsThis, 0, xsVar(1), xsVar(2));

#### xsTest

Like an `if` clause in ECMAScript, the `xsTest` macro takes a value of any type and determines whether it is true or false. This macro applies the same rules as in ECMAScript (per the ECMA-262 specification, section 12.5).

```javascript
xsBooleanValue xsTest(xsSlot theValue)
xsBooleanValue xsmcTest(xsSlot theValue)
```

| |
| --- |
| `theValue` | 

> The value to test


| |
| --- |
| Returns | 

> `true` if the value is true, `false` otherwise


#####In ECMAScript:

	if (foo) {}

#####In C:

<div class="ccode"></div>

	if (xsTest(xsGet(xsGlobal, xsID_foo)) {}
	
#### xsEnumerate

Use the `xsEnumerate` macro to get an iterator for enumerable instance properties. The iterator provides `next`, `value` and `done` functions to iterate the properties. 

`xsSlot xsEnumerate(xsSlot theObject)`

| |
| --- |
| `theObject` | 

> A reference to the object that has enumerable properties


| |
| --- |
| Returns | 

> A slot containing the iterator

#####In ECMAScript:

	rectangle = { x:0, y:0, width:200, height:100 };
	for (let prop in rectangle)
		trace(`${prop}: ${rectangle[prop]}\n`);

#####In C:

<div class="ccode"></div>

	xsVars(5);
	xsVar(0) = xsGet(xsGlobal, xsID_rectangle);
	xsVar(1) = xsEnumerate(xsVar(0));
	for (;;) {
		xsVar(2) = xsCall0(xsVar(1), xsID("next"));
		if (xsTest(xsGet(xsVar(2), xsID("done"))))
			break;
		xsVar(3) = xsGet(xsVar(2), xsID("value"));
		xsVar(4) = xsGetAt(xsVar(0), xsVar(3));
		xsTrace(xsToString(xsVar(3)));xsTrace(": ");
		xsTrace(xsToString(xsVar(4)));xsTrace("\n");
	}

<a id="xsvars"></a>
### Arguments and Variables

The XS runtime virtual machine uses a heap and a stack of slots. With XS in C, you can access stack slots directly and heap slots indirectly, through references.

When a C callback is executed, the stack contains its argument slots, its `this` slot, and its result slot, but no variable slots. To use variable slots, you have to reserve them on the stack with the `xsVars` or `xsmcVars` macros. The `xsVars` macro can only be used once at the beginning of the callback execution. The `xsmcVars` macro can be used multiple times within a callback. Using `xsmcVars`, the callback can use a different number of variables in different branches of the code, to reduce the XS stack size.

```javascript
void xsVars(xsIntegerValue theCount)
void xsmcVars(xsIntegerValue theCount)
```

| |
| --- |
| `theCount` | 

> The number of variable slots to reserve

Argument and variable slots are accessed and assigned by index. An exception is thrown if the index is invalid.

Initially:

- The argument slots are the parameter slots passed to the function or constructor.

- If the callback is a function, the `this` slot refers to the instance being called and the result slot is undefined.

- If the callback is a constructor, the `this` and result slots refer to the instance being created.

- The variable slots are undefined.

Scripts can call a constructor as a function or a function as a constructor. To find out whether the C callback is executed as a constructor or as a function, you can check whether the result slot is initially undefined.

```javascript
xsSlot xsArgc
int xsmcArgc
```

| |
| --- |
| Returns |

> An integer slot that contains the number of arguments

```javascript
xsSlot xsArg(xsIntegerValue theIndex)
```

| |
| --- |
| `theIndex` | 

> The index of the argument, from 0 to `xsArgc-1`

| |
| --- |
| Returns | 

> The argument slot

`xsSlot xsThis`

| |
| --- |
| Returns | 

> The `this` slot

`xsSlot xsResult`

| |
| --- |
| Returns | 

> The result slot

`xsSlot xsVarc`

| |
| --- |
| Returns | 

> An integer slot that contains the number of variables

`xsSlot xsVar(xsIntegerValue theIndex)`

| |
| --- |
| `theIndex` | 

> The index of the variable, from 0 to `xsVarc-1`

| |
| --- |
| Returns | 

> The variable slot

Usually you access the argument, `this`, result, and variable slots but you assign only the result and variable slots. Whatever is in the result slot at the end of the callback execution is returned to scripts by the function or constructor.

In the C example in this section (and the next one), `xsMachine` is the virtual machine structure, as shown in the section [Machine](#machine).

#####In ECMAScript:

	function foo() {
		var c, i, s;
		c = arguments.length;
		s = "";
		for (i = 0; i < c; i++)
			s = s.concat(arguments[i]);
		return s;
	}


#####In C:

<div class="ccode"></div>

	void xs_foo(xsMachine* the) {
		xsIntegerValue c, i;
		xsVars(1);
		c = xsToInteger(xsArgc));
		xsVar(0) = xsString("");
		for (i = 0; i < c; i++)
			xsVar(0) = xsCall1(xsVar(0), xsID_concat, xsArg(i));
		xsResult = xsVar(0);
	}

### Garbage Collector

When the XS runtime needs to allocate slots and there is not enough memory, it automatically deletes unused slots. The runtime garbage collector uses a mark and sweep algorithm.  To force the runtime to delete unused  slots, you can use the `xsCollectGarbage` macro.

`void xsCollectGarbage()`

If you store slots in memory that is no managed by the garbage collector, such as a C global or a C allocated structure, use the `xsRemember` and `xsForget` macros to inform the runtime.

`void xsRemember(xsSlot theSlot)`

| |
| --- |
| `theSlot` | 

> The slot to remember

`void xsForget(xsSlot theSlot)`

| |
| --- |
| `theSlot` | 

> The slot to forget

`xsRemember` links and `xsForget` unlinks a slot to and from a chain of slots which the garbage collector scans to mark the slots that the C global or the C allocated structure references.

Use `xsAccess` to get the value of a slot previously linked to the chain of slots.

`xsSlot xsAccess(xsSlot theSlot)`

| |
| --- |
| `theSlot` | 

> The slot to access

| |
| --- |
| Returns | 

> The value of the slot

#####In C:

<div class="ccode"></div>

	xsSlot gFooSlot;
	void xsSetupFoo(xsMachine* the) {
		gFooSlot = xsThis;
		xsRemember(gFooSlot);
	}
	void xsInvokeFoo(xsMachine* the) {
		xsVars(2);
		xsVar(0) = xsAccess(gFooSlot);
		xsVar(1) = xsString("message");
		xsCall1(xsVar(0), xsID_invoke, xsVar(1));
	}
	void xsCleanupFoo(xsMachine* the) {
		xsForget(gFooSlot);
	}

The garbage collector is enabled by default. Use `xsEnableGarbageCollection` to enable or disable the garbage collector.
 
`xsSlot xsEnableGarbageCollection(xsBooleanValue enable)`

| |
| --- |
| `enable` | 

> Set `true` to enable garbage collection or `false` to disable garbage collection

### Exceptions

To handle exceptions in C, the XS runtime uses `setjmp`, `longjmp`, and a chain of `jmp_buf` buffers, defined as follows:

<!--From CR: On the web page, only "stack" is highlighted (blue) in the following code and similar code in this doc; is something wrong?-->

<div class="ccode"></div>

	typedef struct xsJumpRecord xsJump
	struct xsJumpRecord {
		jmp_buf buffer;
		xsJump* nextJump;
		xsSlot* stack;
		xsSlot* frame;
	};

However, you do not need to use this directly, because XS in C defines macros for throwing and catching exceptions.

To throw an exception, use the `xsThrow` macro.

`void xsThrow(xsSlot theException)`

| |
| --- |
| `theException` | 

> The exception slot

The `xsThrow` macro assigns the current exception; the `xsException` macro accesses the current exception.

`xsSlot xsException`

| |
| --- |
| Returns | 

> The exception slot

As shown in the following example, the `xsTry` and `xsCatch` macros are used together to catch exceptions. If you catch an exception in your C callback and you want to propagate the exception to the script that calls your function or constructor, throw the exception again.

#####In ECMAScript:

	{
		try {
			/* Exception thrown here ... */
		}
		catch(e) {
			/* ... is caught here. */	
			throw e
	 	}
	}

#####In C:

<div class="ccode"></div>

	 {
		xsTry {
			/* Exception thrown here ... */
		}
		xsCatch {
			/* ... is caught here. */
			xsThrow(xsException)
		}
	} 

### Errors

Exceptions may be thrown by C callbacks. C callbacks are often provide the interface between scripts and systems. Many system calls can fail, and they have a way to return an error to the application which can be propegated as an exception.

For specific errors, the XS runtime provides error types and prototypes.

	enum {
		XS_NO_ERROR = 0,
		XS_UNKNOWN_ERROR,
		XS_EVAL_ERROR,
		XS_RANGE_ERROR,
		XS_REFERENCE_ERROR,
		XS_SYNTAX_ERROR,
		XS_TYPE_ERROR,
		XS_URI_ERROR,
		XS_ERROR_COUNT
	};
	
XS in C defines the following macros to throw specific exceptions.

	void xsUnknownError(...)
	void xsEvalError(...)
	void xsRangeError(...)
	void xsReferenceError(...)
	void xsSyntaxError(...)
	void xsTypeError(...)
	void xsURIError(...)

| |
| --- |
| ... | 
> The message and optional arguments to display when throwing the exception

#####In C:

<div class="ccode"></div>

	xpt2046 xpt = calloc(1, sizeof(xpt2046Record));
	if (!xpt) xsUnknownError("out of memory");
	
	if (strlen(string) > MAXNAMESIZE)
		xsRangeError("name too long: %s", string);

	char *slash = strrchr(path, '/');
	if (!slash)
		xsURIError("No path");

The `xsErrorPrintf` macro is a shortcut for `xsUnknownError` when only a message parameter is required.

	xsErrorPrintf(xsStringValue theMessage)
	xsUnknownError("%s", theMessage)	
| |
| --- |
| theMessage | 
> The message to display when throwing the exception

#####In C:

	if (rotation != requestedRotation)
		xsErrorPrintf("not configured for requested rotation");
		
### Debugger

XS in C provides two macros to help you debug your C callbacks.

The `xsDebugger` macro is equivalent to the ECMAScript `debugger` keyword.

`void xsDebugger()`

The `xsTrace` macro is equivalent to the global `trace` function.

`void xsTrace(xsStringValue theMessage)`

| |
| --- |
| `theMessage` | 

> The message to log in the debugger

#####In ECMAScript:

<div class="ccode"></div>

	debugger;
	trace("Hello xsbug!\n");

#####In C:

<div class="ccode"></div>

	 xsDebugger();
	 xsTrace("Hello xsbug!\n");


<a id="machine"></a>
##Machine

The main structure of the XS runtime is its virtual machine, which is what parses, compiles, links, and executes scripts. A virtual machine is an opaque structure though some members of the structure are available to optimize the macros of XS in C; you never need to use them directly.

<div class="ccode"></div>

	typedef struct xsMachineRecord xsMachine
	struct xsMachineRecord {
		xsSlot* stack;
		xsSlot* scope;
		xsSlot* frame;
		xsIndex* code;
		xsSlot* stackBottom;
		xsSlot* stackTop;
		xsSlot* stackPrototypes;
		xsJump* firstJump;
	};

A single machine does not support multiple threads. To work with multiple threads, create one XS runtime machine for each thread, with the host optionally providing a way for the machines to communicate. 

<a id="machine-allocation"></a>
###Machine Allocation

To use the XS runtime you have to create a machine with the `xsCreateMachine` macro, allocating memory for it as required. Its parameters are:

-  A structure with members that are essentially parameters specifying what to allocate for the machine. Pass `NULL` if you want to use the defaults. 

<div class="ccode indentCode"></div>

	typedef struct {
		xsIntegerValue initialChunkSize;
		xsIntegerValue incrementalChunkSize;
		xsIntegerValue initialHeapCount;
		xsIntegerValue incrementalHeapCount;
		xsIntegerValue stackCount;
		xsIntegerValue keyCount;
		xsIntegerValue nameModulo;
		xsIntegerValue symbolModulo;
		xsIntegerValue staticSize;
	} xsCreation;

- The name of the machine

- A context you can set and get in your callbacks (as discussed in the next section). Pass `NULL` if you do not want an initial context.

`xsMachine* xsCreateMachine(xsCreation* theCreation, xsStringValue theName, void* theContext)`

| |
| --- |
| `theCreation` | 

> The parameters of the machine

| |
| --- |
| `theName` | 

> The name of the machine as a string

| |
| --- |
| `theContext` | 

> The initial context of the machine, or `NULL`

| |
| --- |
| Returns | 

> A machine if successful, otherwise `NULL`

Regarding the parameters of the machine that are specified in the `xsCreation` structure:

- A machine manages strings and bytecodes in chunks. The initial chunk size is the initial size of the memory allocated to chunks. The incremental chunk size tells the runtime how to expand the memory allocated to chunks. 

- A machine uses a heap and a stack of slots. The initial heap count is the initial number of slots allocated to the heap. The incremental heap count tells the runtime how to increase the number of slots allocated to the heap. The stack count is the number of slots allocated to the stack.

- The symbol count is the number of symbols the machine will use. The symbol modulo is the size of the hash table the machine will use for symbols. A symbol binds a string value and an identifier; see [`xsID`](#xs-id).

When you are done with a machine, you free it with the `xsDeleteMachine` macro. The destructors of all the host objects are executed, and all the memory allocated by the machine is freed.

`void xsDeleteMachine(xsMachine* the)`

| |
| --- |
| `the` | 

> A machine

The `xsDeleteMachine` macro is one of a number of macros described in this document that have an explicit machine parameter named `the`, for which the value returned by `xsCreateMachine` is passed. (The other such macros are `xsGetContext`, `xsSetContext`, `xsBeginHost`, and `xsEndHost`.) Only those macros have an explicit `the` parameter because they are the only ones that can be used outside a callback and cannot throw exceptions. Callbacks must name their machine parameter `the` because all other macros have an implicit parameter named `the`; the primary reason for this convention is terseness, but it also emphasizes the fact that these other macros can be used only inside a callback and can throw exceptions.

The following example illustrates the use of `xsCreateMachine` and `xsDeleteMachine`. The `xsMainContext` function called in the example is defined in the next section.


#####Example

<div class="ccode"></div>

	int main(int argc, char* argv[]) 
	{
		xsCreation aCreation = {
			128 * 1024 * 1024, 	/* initialChunkSize */
			16 * 1024 * 1024, 	/* incrementalChunkSize */
			4 * 1024 * 1024, 	/* initialHeapCount */
			1 * 1024 * 1024,		/* incrementalHeapCount */
			1024,					/* stack count */
			2048+1024,			/* key count */
			1993,					/* name modulo */
			127						/* symbol modulo */
		};
		xsMachine* aMachine;

		aMachine = xsCreateMachine(&aCreation, "machine", NULL);
		if (aMachine) {
			xsMainContext(aMachine, argc, argv);
			xsDeleteMachine(aMachine);
		}
		else
			fprintf(stderr, "### Cannot allocate machine\n");
		return 0;
	}

###Context

The machine will call your C code primarily through callbacks. In your callbacks, you can set and get a *context*: a pointer to an area where you can store and retrieve information for the machine.

`void xsSetContext(xsMachine* the, void* theContext)`

| |
| --- |
| `the` | 

> A machine

| |
| --- |
| `theContext` | 

> A context


`void* xsGetContext(xsMachine* the)`

| |
| --- |
| `the` | 

> A machine

| |
| --- |
| Returns | 

> A context

The following code shows a context being set in the `xsMainContext` function, which was called in the preceding section's example.

##### Example

<div class="ccode"></div>

	typedef struct {
		int argc;
		char** argv;
	} xsContext;

	void xsMainContext(xsMachine* theMachine, int argc, char* argv[])
	{
		xsContext* aContext;

		aContext = malloc(sizeof(xsContext));
		if (aContext) {
			aContext->argc = argc;
			aContext->argv = argv;
			xsSetContext(theMachine, aContext);
			xsSetContext(theMachine, NULL);
			free(aContext);
		}
		else
			fprintf(stderr, "### Cannot allocate context\n");
	}

###Host

This section describes the host-related macros of XS in C (see Table 2). The example code that uses these macros is shown after the last macro it uses has been described. (The remaining two macros do not enter into the sample application.)

**Table 2.** Host-Related Macros

<table class="normalTable">
  <tbody>
    <tr>
      <th scope="col">Macro</th>
      <th scope="col">Description</th>
    </tr>
    <tr>
      <td>
        <p><code>xsNewHostFunction</code></p>
        <p><code>xsNewHostConstructor</code></p>
      </td>
      <td>Creates a host function or host constructor</td>
    </tr> 
    <tr>
      <td><code>xsNewHostObject</code></td>
      <td>Creates a host object</td>
    </tr> 
    <tr>
      <td>
        <p><code>xsGetHostData, xsmcGetHostData</code></p>
        <p><code>xsSetHostData, xsmcSetHostData</code></p>
      </td>
      <td>Gets or sets the data in a host object</td>
    </tr> 
    <tr>
      <td>
        <p><code>xsGetHostChunk, xsmcGetHostChunk</code></p>
        <p><code>xsSetHostChunk, xsmcSetHostChunk</code></p>
      </td>
      <td>Gets or sets the data as a chunk in a host object</td>
    </tr> 
    <tr>
      <td><code>xsSetHostDestructor</code></td>
      <td>Sets the destructor for a host object</td>
    </tr> 
    <tr>
      <td>
        <p><code>xsBeginHost</code></p>
        <p><code>xsEndHost</code></p>
      </td>
      <td>Used together to set up and clean up a stack frame, so that you can use all the macros of XS in C in between</td>
    </tr> 
  </tbody>
</table>

#### xsNewHostFunction and xsNewHostConstructor

A *host function* is a special kind of function, onw whose implementation is in C rather than ECMAScript. For a script, a host function is just like a function; however, when a script invokes a host function, a C callback is executed. The same is true for *host constructors*, which are constructors implemented in C.

<div class="ccode"></div>

	typedef void (*xsCallback)(xsMachine* the);

To create a host function, use the `xsNewHostFunction` macro.

`xsSlot xsNewHostFunction(xsCallback theCallback, xsIntegerValue theLength);`

| |
| --- |
| `theCallback` | 

> The callback to execute

| |
| --- |
| `theLength` | 

> The number of parameters expected by the callback

| |
| --- |
| Returns | 

> A reference to the new host function


To create a host constructor, use the `xsNewHostConstructor` macro.

`xsSlot xsNewHostConstructor(xsCallback theCallback,`
`	xsIntegerValue theLength, xsSlot thePrototype)`

| |
| --- |
| `theCallback` | 

> The callback to execute

| |
| --- |
| `theLength` | 

> The number of parameters expected by the callback

| |
| --- |
| `thePrototype` | 

> A reference to the prototype of the instance to create

| |
| --- |
| Returns | 

> A reference to the new host constructor


#### xsNewHostObject

A *host object* is a special kind of object with data that can be directly accessed only in C. The data in a host object is invisible to scripts. 

When the garbage collector is about to get rid of a host object, it executes the host object's destructor, if any. No reference to the host object is passed to the destructor: a destructor can only destroy data.

<div class="ccode"></div>

	typedef void (xsDestructor)(void* theData);

To create a host object, use the `xsNewHostObject` macro. Pass the host object's destructor, or `NULL` if it does not need a destructor. 

`xsSlot xsNewHostObject(xsDestructor theDestructor)`

| |
| --- |
| `theDestructor` | 

> The destructor to be executed by the garbage collector

| |
| --- |
| Returns | 

> A reference to the new host object


#### xsGetHostData and xsSetHostData

To get and set the data of a host object, use the `xsGetHostData` and `xsSetHostData` macros. Both throw an exception if the `theThis` parameter does not refer to a host object.

```javascript
void* xsGetHostData(xsSlot theThis)
void* xsmcGetHostData(xsSlot theThis)
```

| |
| --- |
| `theThis` | 

> A reference to a host object

| |
| --- |
| Returns | 

> The data

```javascript
void xsSetHostData(xsSlot theThis, void* theData)
void xsmcSetHostData(xsSlot theThis, void* theData)
```

| |
| --- |
| `theThis` | 

> A reference to a host object

| |
| --- |
| `theData` | 

> The data to set

#### xsGetHostChunk and xsSetHostChunk

To get and set the data of a host object as a chunk, use the `xsGetHostChunk` and `xsSetHostChunk` macros. Both throw an exception if the `theThis` parameter does not refer to a host object. Like the memory used by ArrayBuffer and String, chunk memory is allocated and managed by the XS runtime; see the [handle](./handle.md) document for details. 

```javascript
void* xsGetHostChunk(xsSlot theThis)
void* xsmcGetHostChunk(xsSlot theThis)
```

| |
| --- |
| `theThis` | 

> A reference to a host object

| |
| --- |
| Returns | 

> The data

```javascript
void xsSetHostChunk(xsSlot theThis, void* theData, xsIntegerValue theSize)
void xsmcSetHostChunk(xsSlot theThis, void* theData, xsIntegerValue theSize)
```

| |
| --- |
| `theThis` | 

> A reference to a host object

| |
| --- |
| `theData` | 

> The data to set or `NULL` to leave the chunk data uninitialized

| |
| --- |
| `theSize` | 

> The size of the data in bytes

Note that an object has either host data or a host chunk but never both. 

#### xsSetHostDestructor

To set the destructor of a host object (or to clear the destructor, by passing `NULL`), use the `xsSetHostDestructor` macro. This macro throws an exception if the `theThis` parameter does not refer to a host object.

`void xsSetHostDestructor(xsSlot theThis, xsDestructor theDestructor)`

| |
| --- |
| `theThis` | 

> A reference to a host object

| |
| --- |
| `theDestructor` | 

> The destructor to be executed by the garbage collector

#### xsBeginHost and xsEndHost

Use the `xsBeginHost` macro to establish a new stack frame and the `xsEndHost` macro to remove it. 

`void xsBeginHost(xsMachine* the)`

| |
| --- |
| `the` | 

> A machine

`void xsEndHost(xsMachine* the)`

| |
| --- |
| `the` | 

> A machine

The `xsBeginHost` macro sets up the stack, and the `xsEndHost` macro cleans up the stack, so that you can use all the macros of XS in C in the block between `xsBeginHost` and `xsEndHost`.

Uncaught exceptions that occur between the calls the `xsBeginHost` and `xsEndHost `do not propagate beyond `xsEndHost`.

##### Example

<div class="ccode"></div>

	long FAR PASCAL xsWndProc(HWND hwnd, UINT m, UINT w, LONG l)
	{
		long result = 0;
		xsMachine* aMachine = GetWindowLongPtr(hwnd, GWL_USERDATA);
		xsBeginHost(aMachine);
		{
			result = xsToInteger(xsCall3(xsGlobal, xsID_dispatch, 
				xsInteger(m), xsInteger(w), xsInteger(l)));
		} 
		xsEndHost(aMachine);
		return result;
	}

### JavaScript `@` language syntax extension

XS provides the `@` language syntax extension to implement JavaScript functions in C. The language extension is only recognized by the XS compiler. This section introduces the language extension with a JavaScript class that implements methods with C functions.

```javascript
class Rectangle @ "xs_rectangle_destructor" {
	constructor(...params) @ "xs_rectangle";

	get x() @ "xs_rectangle_get_x";
	set x() @ "xs_rectangle_set_x";
	get y() @ "xs_rectangle_get_y";
	set y() @ "xs_rectangle_set_y";
	get w() @ "xs_rectangle_get_w";
	set w() @ "xs_rectangle_set_w";
	get h() @ "xs_rectangle_get_h";
	set h() @ "xs_rectangle_set_h";

	contains(x, y) @ "xs_rectangle_contains";

	union(r) @ "xs_rectangle_union";
};

export default Rectangle;
```
The `Rectangle` class is completely implemented in C using callbacks specified by `@` functions. For example, the `contains` method is implemented by the `xs_rectangle_contains` C function. The C functions use XS in C macros to access properties, host data, and return results. JavaScript applications import the `Rectangle` class and access the methods.

```javascript
import Rectangle from "rectangle";

let r1 = new Rectangle(0, 0, 200, 100);
let r2 = new Rectangle(20, 40, 300, 50);
let r3 = new Rectangle();
r3.union(r1, r2);
```

The `Rectangle` constructor `xs_rectangle` function stores the parameters in a host chunk. The constructor accepts either a single `Rectangle` instance or the individual `x`, `y`, `w` and `h` values. The function uses the `xsmcArgc` macro to count the function parameters and the `xsmcIsInstanceOf` macro to determine if the first parameter is an object.

```javascript
typedef struct {
	int x;
	int y;
	int w;
	int h;
} xsRectangleRecord, *xsRectangle;

void xs_rectangle(xsMachine *the)
{
	xsRectangleRecord r;
	if (xsmcArgc == 0) {
		r.x = r.y = r.w = r.h = 0;
	}
	else if (xsmcIsInstanceOf(xsArg(0), xsObjectPrototype)) {
		xsRectangle r1 = xsmcGetHostChunk(xsArg(0));
		r = *r1;
	}
	else {
		r.x = xsmcToInteger(xsArg(0));
		r.y = xsmcToInteger(xsArg(1));
		r.w = xsmcToInteger(xsArg(2));
		r.h = xsmcToInteger(xsArg(3));
	}
	xsmcSetHostChunk(xsThis, &r, sizeof(r));
}
```
The destructor function `xs_rectangle_destructor` is called when the object instance is deleted or garbage collected. Any memory or resources allocated by the instance should be freed in the destructor. Because the XS runtime manages host chunk memory, the destructor doesn't need to dispose the chunk.

```javascript
void xs_rectangle_destructor(void *data)
{
}
```
The `Rectangle` class provides getters and setters for class properties.

```javascript
	get x() @ "xs_rectangle_get_x";
	set x() @ "xs_rectangle_set_x";
```

The `get` functions read the corresponding field from the host chunk and return the property to the caller by setting `xsResult`. The `set` functions store the value provided into the host chunk.

```javascript
void xs_rectangle_get_x(xsMachine *the)
{
	xsRectangle r = xsmcGetHostChunk(xsThis);
	xsmcSetInteger(xsResult, r->x);
}

void xs_rectangle_set_x(xsMachine *the)
{
	xsRectangle r = xsmcGetHostChunk(xsThis);
	r->x = xsmcToInteger(xsArg(0));
}
```
The `union` method returns the union of all the rectangles passed to the function. The `xs_rectangle_union` function uses the `xsmcArgc` macro to count the number or `Rectangle` instances. The union result is stored back into calling instance's host chunk. JavaScript applications read the result rectangle properties using the `get *()` methods.

```javascript
void xs_rectangle_union(xsMachine *the)
{
	xsIntegerValue i, argc;
	xsRectangle r, r0 = xsmcGetHostChunk(xsThis);
	xsRectangleRecord rUnion;
	r = r0;
	for (i = 0; i < argc; ++i) {
		Union(&rUnion, r, xsmcGetHostChunk(xsArg(i)));
		r = &rUnion;
	}
	*r0 = rUnion;
}
```

<a id="glossary"></a>
## Glossary

##### constructor

In ECMAScript, a function that has a `prototype` property and that the `new` operator invokes to build an instance. The value of the `prototype` property becomes the prototype of the instances that the constructor builds.

##### context

A pointer to an area where you can store and retrieve information for the XS runtime virtual machine in your callbacks.

##### direct slot

One of the slot types that correspond to the ECMAScript primitive types (undefined, null, boolean, number, string, and symbol), plus an integer and stringx slot provided as an optimization.

##### ECMAScript

An object-oriented, prototype-based language for implementing application logic and control.

##### host

In ECMAScript terminology, an application that uses the XS runtime.

##### host constructor

In XS, a constructor whose implementation is in C rather than ECMAScript.

##### host function

In XS, a function whose implementation is in C rather than ECMAScript.

##### host object

In XS, an object with data that can be directly accessed only in C.

##### indirect slot

A type of slot that contains a reference to an instance of an object, function, array, and so on; corresponds to the ECMAScript `reference` type.

##### instance

An object that inherits properties from another object, which is called its *prototype*.

##### property

In ECMAScript, a value accessed by name within an object (in contrast to items accessed by index within an array); in XS in C, a slot accessed by index within an object (just as an item is accessed by index within an array).

##### prototype

An object from which another object (called an instance) inherits properties.

##### sandbox

An environment that is restricted to prevent untrusted code from harming the device on which the code is running. The sandbox for XS application scripts includes the standard features defined in the ECMAScript specification plus additional features as defined and permitted by the XS modules.

##### slot

An opaque structure in which everything in the XS runtime is stored, and which is manipulated only through XS in C.

##### XS

A toolkit, consisting of a runtime library and a command-line tool, that is designed for developing standards-based, networked, interactive multimedia applications (GUI-based runtimes) or command-line tools for various devices. See also [**XS runtime**](#xsruntime) and [**xsc**](#xsc).

<a name="xsruntime"></a>
##### XS runtime

The runtime library part of XS.

##### XS in C

The C interface of the XS runtime.

##### xsbug

The XS debugger, used to debug applications, modules, and scripts.

<a name="xsc"></a>
##### xsc

The command-line tool part of XS. It compiles JavaScript files into XS binary files containing symbols and bytecodes, which is executed by the XS virtual machine that is contained within the XS runtime.

<!-- TBD:
	- Document xsCall*_noResult, xsmcCall_noResult
	- Document xsNewHostConstructorObject, xsNewHostFunctionObject
	- Document xsmcNewHostInstance
	- Document: xsReference
-->

## License
    Copyright (c) 2016-2018  Moddable Tech, Inc.
 
    This file is part of the Moddable SDK Runtime.
  
    The Moddable SDK Runtime is free software: you can redistribute it and/or modify
    it under the terms of the GNU Lesser General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.
  
    The Moddable SDK Runtime is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU Lesser General Public License for more details.
  
    You should have received a copy of the GNU Lesser General Public License
    along with the Moddable SDK Runtime.  If not, see <http://www.gnu.org/licenses/>.

    This file incorporates work covered by the following copyright and  
    permission notice:  

        Copyright (C) 2010-2016 Marvell International Ltd.
        Copyright (C) 2002-2010 Kinoma, Inc.
 
        Licensed under the Apache License, Version 2.0 (the "License");
        you may not use this file except in compliance with the License.
        You may obtain a copy of the License at
 
         http://www.apache.org/licenses/LICENSE-2.0
 
        Unless required by applicable law or agreed to in writing, software
        distributed under the License is distributed on an "AS IS" BASIS,
        WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        See the License for the specific language governing permissions and
        limitations under the License.