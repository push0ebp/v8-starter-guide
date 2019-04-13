# V8

## Build
[Building V8 from source · V8](https://v8.dev/docs/build)
```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=`pwd`/depot_tools:"$PATH"
fetch v8 
cd v8 
git checkout c895a23
gclient sync
build/install-build-deps.sh # only Linux

tools/dev/gm.py x64.debug

OR

tools/dev/v8gen.py x64.debug #generate build option template
ninja -C out.gn/x64.debug #build
```
`depot_tools`라는 빌드 도구를 설치하고 환경변수 `PATH`에 추가한다.
`fetch` 커맨드로 v8 source를 다운 받는다.
디버깅할 버전으로 `checkout` 하고 `sync`한다. 
**linux**일 경우 `install-build-deps.sh`로 dependency를 설치한다.
**osx**일 경우 `Xcode CommandLineTools`가 아닌 `Xcode.app`을 설치해야한다.  `Xcode-select -print-path`시 `.app/Contents/Developer`가 있어야함
High Sierra에서 build할시  10.14 SDK가  v8 기준인 10.12 SDK와 맞지 않다고 에러가 출력되는데 스크립트에서 `!=`부분을 수정해주고 다시 빌드 하면된다.

바로  빌드하고 싶으면 `gm.py`로 바로 빌드하면 된다.
build option 수정하고 싶다면 `v8gen.py`로 option template을 생성후 `out.gn/args.gn`수정후 `ninja`로 드드

### v8gen Example

```bash
tools/dev/v8gen.py x64.debug -- v8_enable_slow_dchecks=false v8_enable_backtrace=true
```
```bash
cat out.gn/x64.debug/args.gn
is_debug = true
target_cpu = "x64"
v8_enable_backtrace = true
v8_enable_slow_dchecks = true
v8_optimized_debug = false

# Additional command-line args:
v8_enable_slow_dchecks=false
v8_enable_backtrace=true
```

## Issue

* debug 모드로 빌드하면 poc 트리거할 때 `DCHECK`,`V8_Dcheck` 에 걸려서 `v8::base::OS::Abort()` 로 `SIGILL`이 발생한다. 
  * `src/base/logging.cc`에 `V8_Dcheck`를 수정하면 된다.
  ```c
  void V8_Dcheck(const char* file, int line, const char* message) {
  //v8::base::g_dcheck_function(file, line, message);
  }
  ```

* DoubleArray OOB 발생할때 `abort: CSA_ASSERT failed: IsFixedDoubleArray(object) [../../src/code-stub-assembler.cc:1932]` Assertion이 잡아낸다.
  * `src/code-stub-assembler.h`:96에 DEBUG조건을 반대로 하여  `CSA_ASSERT`함수들을 변경한다. 
  ```c
  void CodeStubAssembler::Check(const BranchGenerator& branch,
  ...
  {
  int i=0;
  if(i) {
  #ifdef DEBUG
    // Only print the extra nodes in debug builds.
    MaybePrintNodeWithName(this, extra_node1, extra_node1_name); //있어야 unused 안뜸
  ...
  #endif
  }
  }
  ```

  * `CSA_ASSERT`등 macro 함수를 바꾸려 했지만 unused 등이 너무 많이 발생하여 Code body를 바꾸도록 하였다.

## Debugging

* pwndbg로 디버깅하면 source code level에서 디버깅이 가능하다. gdb는 start 쳐야함 (feat. Juno)
* debug information obj 폴더에 있으니 삭제하면 안된다.
* osx는 debug info path가 절대경로로 되어있다. linux는 상대경로로 되어있음.
* `v8::D8Console::Log`에 breakpoint를 걸고 `args.values_`로 포인터 본후 `job`을 사용하면 된다.
* `%DebugPrint()`를 사용해도된다.
  * `./d8 --allow-natives-syntax` 옵션을 주고 실행해야 debug function 을 사용할수 있다.
  * DEBUG build mode에서만 object info가 나온다. 
  * RELEASE에서도 사용이 가능하지만 object info가 나오지 않는다.

## Javascript Object

### IntArray vs DoubleArray

javascript에서 64bit integer형은 없다.  이렇기 때문에 8 byte를 write할때는 4byte씩 2개로 나누어 write 해야한다.

우리가 흔히 아는 `int[]`와 `long[]`은 v8에서 구현이 다르다. 4byte integer는  `FixedArray`로 만들어지며 이런 형태로 저장되게 된다. (아래에 디버깅 결과를 첨부하였다.)

```
0x6b0f8e8d440:	0x1122334400000000	0x5566778800000000 

0x6b0f8e8d440:	0x00	0x00	0x00	0x00	0x44	0x33	0x22	0x11
0x6b0f8e8d448:	0x00	0x00	0x00	0x00	0x88	0x77	0x66	0x55
```

0~0x7ffffffff 까지는 `FixedArray`로 저장되며 0x80000000부터는 `FixedDoubleArray`로 저장된다. 

```javascript
fixedArray = [0x7FFFFFFF] //FixedArray
fixedDoubleArray = [0x80000000] //FixedDoubleArray
```

정수형으로 저장하든 상관없이 2^31부터는 v8 내부에선 `double`로 관리한다. 

4 byte단위로 값을 쓸수 없으므로  int가 아닌 **double**로 접근하여 8 byte 값을 read/write 해야한다. 



### DataView

`DataView`는 buffer를 가지고 있다. `ArrayBuffer`가 constructor의 인자로 들어간다. 

DataView를 쓰는 이유는

* Low하게 접근할수 있다.
* Pointer/`buffer`를 바꿀수 있다.
* `byte_offset`과 `byte_length`가 존재하여 Memory Read/Write를 쉽게 할수 있다.



### BoxedArray vs UnboxedArray

BoxedArray는 Object로 이루어진 Array이고 UnboxedArray는 primitive한 int로 이루어진 array이다.

이 둘을 응용하여 포인터를 바꿈으로써 **read/write primitive**를 유도할수 있다.



## Debug Javscript Object

### FixedArray

```javascript
console.log([0x11223344, 0x55667788])
```

Object


```

pwndbg> job *args.values_
0x6b0f8e8d481: [JSArray]
 - map: 0x119501782521 <Map(PACKED_SMI_ELEMENTS)> [FastProperties]
 - prototype: 0x17e1e7105691 <JSArray[0]>
 - elements: 0x6b0f8e8d431 <FixedArray[2]> [PACKED_SMI_ELEMENTS (COW)]
 - length: 2
 - properties: 0x73a9c982251 <FixedArray[0]> {
    #length: 0x73a9c9ad459 <AccessorInfo> (const accessor descriptor)
 }
 - elements: 0x6b0f8e8d431 <FixedArray[2]> {
           0: 287454020
           1: 1432778632
 }

pwndbg> x/40gx 0x6b0f8e8d481-1
0x6b0f8e8d480:	0x0000119501782521	0x0000073a9c982251
0x6b0f8e8d490:	0x000006b0f8e8d431	0x0000000200000000
0x6b0f8e8d4a0:	0x0000233125184321	0x000017e1e71275c9
```

map

### Map

size : 0x50 추정

```

pwndbg> job 0x119501782521
0x119501782521: [Map]
 - type: JS_ARRAY_TYPE
 - instance size: 32
 - inobject properties: 0
 - elements kind: PACKED_SMI_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - back pointer: 0x73a9c9822e1 <undefined>
 - instance descriptors (own) #1: 0x17e1e7105941 <DescriptorArray[5]>
 - layout descriptor: (nil)
 - transitions #1: 0x17e1e7105801 <TransitionArray[4]>Transition array #1:
     0x73a9c9acca9 <Symbol: (elements_transition_symbol)>: (transition to HOLEY_SMI_ELEMENTS) -> 0x1195017825c1 <Map(HOLEY_SMI_ELEMENTS)>

 - prototype: 0x17e1e7105691 <JSArray[0]>
 - constructor: 0x17e1e71052d1 <JSFunction Array (sfi = 0x73a9c997b31)>
 - dependent code: 0x73a9c982251 <FixedArray[0]>
 - construction counter: 0
pwndbg> x/40gx 0x119501782521-1
	0x119501782520:	0x0000233125182251	0x0100042414040404
	0x119501782530:	0x00000000092007ff	0x000017e1e7105691
	0x119501782540:	0x000017e1e71052d1	0x000017e1e7105801
	0x119501782550:	0x000017e1e7105941	0x0000000000000000
	0x119501782560:	0x0000073a9c982251	0x000017e1e7105979
```

elements


```

pwndbg> job 0x6b0f8e8d431
0x6b0f8e8d431: [FixedArray]
 - map: 0x233125182661 <Map(HOLEY_ELEMENTS)>
 - length: 2
           0: 287454020
           1: 1432778632
pwndbg> x/40bx 0x6b0f8e8d431-1
	0x6b0f8e8d430:	0x61	0x26	0x18	0x25	0x31	0x23	0x00	0x00
	0x6b0f8e8d438:	0x00	0x00	0x00	0x00	0x02	0x00	0x00	0x00
	0x6b0f8e8d440:	0x00	0x00	0x00	0x00	0x44	0x33	0x22	0x11
	0x6b0f8e8d448:	0x00	0x00	0x00	0x00	0x88	0x77	0x66	0x55
	0x6b0f8e8d450:	0x61	0x26	0x18	0x25	0x31	0x23	0x00	0x00
pwndbg> x/40gx 0x6b0f8e8d431-1
	0x6b0f8e8d430:	0x0000233125182661	0x0000000200000000 map | length
	0x6b0f8e8d440:	0x1122334400000000	0x5566778800000000 arr[0] | arr[1]
	0x6b0f8e8d450:	0x0000233125182661	0x0000000400000000
	0x6b0f8e8d460:	0x0000073a9c985521	0x000006b0f8e8c711
	0x6b0f8e8d470:	0x0000000000000000	0xffffffff00000000
	0x6b0f8e8d480:	0x0000119501782521	0x0000073a9c982251
	0x6b0f8e8d490:	0x000006b0f8e8d431	0x0000000200000000
	0x6b0f8e8d4a0:	0x0000233125184321	0x000017e1e71275c9
```

elements.map

```
pwndbg> job 0x233125182661
0x233125182661: [Map]
 - type: FIXED_ARRAY_TYPE
 - instance size: 0
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - non-extensible
 - back pointer: 0x73a9c9822e1 <undefined>
 - instance descriptors (own) #0: 0x73a9c982231 <DescriptorArray[2]>
 - layout descriptor: (nil)
 - prototype: 0x73a9c982201 <null>
 - constructor: 0x73a9c982201 <null>
 - dependent code: 0x73a9c982251 <FixedArray[0]>
 - construction counter: 0
pwndbg> x/40gx 0x233125182661-1
	0x233125182660:	0x0000233125182251	0x0c0000b20b000000
	0x233125182670:	0x00000000002003ff	0x0000073a9c982201
	0x233125182680:	0x0000073a9c982201	0x0000000000000000
	0x233125182690:	0x0000073a9c982231	0x0000000000000000
	0x2331251826a0:	0x0000073a9c982251	0x0000000000000000
	0x2331251826b0:	0x0000233125182251	0x0d0000b40b000000
```



### FixedDoubleArray

```javascript
console.log([1.1, 0x1122334455667788])
```

object

```
pwndbg> job *args.values_
0x11e14ab0d4c1: [JSArray]
 - map: 0x34c3f5382611 <Map(PACKED_DOUBLE_ELEMENTS)> [FastProperties]
 - prototype: 0x1493f3e85691 <JSArray[0]>
 - elements: 0x11e14ab0d4f1 <FixedDoubleArray[2]> [PACKED_DOUBLE_ELEMENTS]
 - length: 2
 - properties: 0x319a2ee82251 <FixedArray[0]> {
    #length: 0x319a2eead459 <AccessorInfo> (const accessor descriptor)
 }
 - elements: 0x11e14ab0d4f1 <FixedDoubleArray[2]> {
           0: 1.1
           1: 1.23461e+18
 }
pwndbg> x/40gx *args.values_
	0x11e14ab0d4c1:	0x51000034c3f53826	0xf10000319a2ee822
	0x11e14ab0d4d1:	0x00000011e14ab0d4	0x2100000002000000
	0x11e14ab0d4e1:	0xe900003235f3c043	0x7100001493f3ea75
	0x11e14ab0d4f1:	0x0000003235f3c02f	0x9a00000002000000
	0x11e14ab0d501:	0x783ff19999999999	0xef43b12233445566
```

map

```
pwndbg> job 0x34c3f5382611
0x34c3f5382611: [Map]
 - type: JS_ARRAY_TYPE //0x424
 - instance size: 32
 - inobject properties: 0
 - elements kind: PACKED_DOUBLE_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - back pointer: 0x34c3f53825c1 <Map(HOLEY_SMI_ELEMENTS)>
 - instance descriptors #1: 0x1493f3e85941 <DescriptorArray[5]>
 - layout descriptor: (nil)
 - transitions #1: 0x1493f3e85881 <TransitionArray[4]>Transition array #1:
     0x319a2eeacca9 <Symbol: (elements_transition_symbol)>: (transition to HOLEY_DOUBLE_ELEMENTS) -> 0x34c3f5382661 <Map(HOLEY_DOUBLE_ELEMENTS)>

 - prototype: 0x1493f3e85691 <JSArray[0]>
 - constructor: 0x1493f3e852d1 <JSFunction Array (sfi = 0x319a2ee97b31)>
 - dependent code: 0x319a2ee82251 <FixedArray[0]>
 - construction counter: 0
pwndbg> x/40gx 0x34c3f5382611-1
	0x34c3f5382610:	0x00003235f3c02251	0x1100042414040404
	0x34c3f5382620:	0x00000000090007ff	0x00001493f3e85691
	0x34c3f5382630:	0x000034c3f53825c1	0x00001493f3e85881
	0x34c3f5382640:	0x00001493f3e85941	0x0000000000000000
	0x34c3f5382650:	0x0000319a2ee82251	0x00001493f3e85871
```

elements

```
pwndbg> job 0x11e14ab0d4f1
0x11e14ab0d4f1: [FixedDoubleArray]
 - map: 0x3235f3c02f71 <Map(HOLEY_DOUBLE_ELEMENTS)>
 - length: 2
           0: 1.1
           1: 1.23461e+18
pwndbg> x/40gx 0x11e14ab0d4f1-1
	0x11e14ab0d4f0:	0x00003235f3c02f71	0x0000000200000000 map | length
	0x11e14ab0d500:	0x3ff199999999999a	0x43b1223344556678 1.1 | 
	0x11e14ab0d510:	0xdeadbeedbeadbeef	0xdeadbeedbeadbeef
```

elements.map

```
0x3235f3c02f71: [Map]
 - type: FIXED_DOUBLE_ARRAY_TYPE
 - instance size: 0
 - elements kind: HOLEY_DOUBLE_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - back pointer: 0x319a2ee822e1 <undefined>
 - instance descriptors (own) #0: 0x319a2ee82231 <DescriptorArray[2]>
 - layout descriptor: (nil)
 - prototype: 0x319a2ee82201 <null>
 - constructor: 0x319a2ee82201 <null>
 - dependent code: 0x319a2ee82251 <FixedArray[0]>
 - construction counter: 0
pwndbg> x/40gx 0x3235f3c02f71-1
	0x3235f3c02f70:	0x00003235f3c02251	0x150000960c000000
	0x3235f3c02f80:	0x00000000082003ff	0x0000319a2ee82201
	0x3235f3c02f90:	0x0000319a2ee82201	0x0000000000000000
	0x3235f3c02fa0:	0x0000319a2ee82231	0x0000000000000000
	0x3235f3c02fb0:	0x0000319a2ee82251	0x0000000000000000
	0x3235f3c02fc0:	0x00003235f3c02251	0x0d00008608000002
```



### DataView

code

```javascript
var arrayBuffer=new ArrayBuffer(0x1234); 
var dataView = new DataView(arrayBuffer); 
dataView.setUint8(0xff); 
dataView.setUint32(0, 0x11223344); 
console.log(dataView);
```

object

```
pwndbg> job *args.values_
0x2b03c2f8d531: [JSDataView]
 - map: 0x155081402ed1 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x19816ee0a661 <Object map = 0x155081402f21>
 - elements: 0x3fef4ff02251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - embedder fields: 2
 - buffer =0x2b03c2f8d4e1 <ArrayBuffer map = 0x155081403d31>
 - byte_offset: 0
 - byte_length: 4660
 - properties: 0x3fef4ff02251 <FixedArray[0]> {}
 - embedder fields = {
    (nil)
    (nil)
 }
pwndbg> x/40gx 0x2b03c2f8d531-1
	0x2b03c2f8d530:	0x0000155081402ed1	0x00003fef4ff02251 map | properties
	0x2b03c2f8d540:	0x00003fef4ff02251	0x00002b03c2f8d4e1 elements | buffer
	0x2b03c2f8d550:	0x0000000000000000	0x0000123400000000 byte_offset|byte_length
	0x2b03c2f8d560:	0x0000000000000000	0x0000000000000000
	0x2b03c2f8d570:	0xdeadbeedbeadbeef	0xdeadbeedbeadbeef
```

map

```

pwndbg> job 0x155081402ed1
0x155081402ed1: [Map]
 - type: JS_DATA_VIEW_TYPE
 - instance size: 64
 - inobject properties: 0
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - back pointer: 0x3fef4ff022e1 <undefined>
 - instance descriptors (own) #0: 0x3fef4ff02231 <DescriptorArray[2]>
 - layout descriptor: (nil)
 - prototype: 0x19816ee0a661 <Object map = 0x155081402f21>
 - constructor: 0x19816ee0a579 <JSFunction DataView (sfi = 0x3fef4ff26b51)>
 - dependent code: 0x3fef4ff02251 <FixedArray[0]>
 - construction counter: 0
pwndbg> x/40gx 0x155081402ed1-1
	0x155081402ed0:	0x000031416a482251	0x0d00043914080808
	0x155081402ee0:	0x00000000082003ff	0x000019816ee0a661
	0x155081402ef0:	0x000019816ee0a579	0x0000000000000000
	0x155081402f00:	0x00003fef4ff02231	0x0000000000000000
	0x155081402f10:	0x00003fef4ff02251	0x0000000000000000
	0x155081402f20:	0x000031416a482251	0x0f00042114040307
```



buffer

```
pwndbg> job 0x2b03c2f8d4e1
0x2b03c2f8d4e1: [JSArrayBuffer]
 - map: 0x155081403d31 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x19816ee12ac9 <Object map = 0x155081403d81>
 - elements: 0x3fef4ff02251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - embedder fields: 2
 - backing_store: 0x558ef0edc650
 - byte_length: 4660
 - neuterable
 - properties: 0x3fef4ff02251 <FixedArray[0]> {}
 - embedder fields = {
    (nil)
    (nil)
 }
pwndbg> x/40gx 0x2b03c2f8d4e1
	0x2b03c2f8d4e1:	0x510000155081403d	0x5100003fef4ff022
	0x2b03c2f8d4f1:	0x0000003fef4ff022	0x5000001234000000
	0x2b03c2f8d501:	0x500000558ef0edc6	0x340000558ef0edc6
	0x2b03c2f8d511:	0x0400000000000012	0x0000000000000000
	0x2b03c2f8d521:	0x0000000000000000	0xd100000000000000
	0x2b03c2f8d531:	0x510000155081402e	0x5100003fef4ff022
	0x2b03c2f8d541:	0xe100003fef4ff022	0x0000002b03c2f8d4
	0x2b03c2f8d551:	0x0000000000000000	0x0000001234000000
	0x2b03c2f8d561:	0x0000000000000000	0xef00000000000000
	0x2b03c2f8d571:	0xefdeadbeedbeadbe	0xefdeadbeedbeadbe
```

map

```
pwndbg> job 0x285cc1983d31
0x285cc1983d31: [Map]
 - type: JS_ARRAY_BUFFER_TYPE
 - instance size: 80
 - inobject properties: 0
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - back pointer: 0x1b4c025822e1 <undefined>
 - instance descriptors (own) #0: 0x1b4c02582231 <DescriptorArray[2]>
 - layout descriptor: (nil)
 - prototype: 0x342ebf92ac9 <Object map = 0x285cc1983d81>
 - constructor: 0x342ebf92929 <JSFunction ArrayBuffer (sfi = 0x1b4c025a4299)>
 - dependent code: 0x1b4c02582251 <FixedArray[0]>
 - construction counter: 0
```



## Arraybuffer

code

```javascript
var arrayBuffer = new ArrayBuffer(0x1234); 
var uint8Array = new Uint32Array(arrayBuffer);
uint8Array[0] = 0x11223344;
uint8Array[1] = 0x55667788;
console.log(arrayBuffer);
```

object

```
pwndbg> job *args.values_
0x2ca728c8d469: [JSArrayBuffer]
 - map: 0x796fdd83d31 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x1f0ae892ac9 <Object map = 0x796fdd83d81>
 - elements: 0x2bf2cca02251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - embedder fields: 2
 - backing_store: 0x55eba87e2650
 - byte_length: 4660
 - neuterable
 - properties: 0x2bf2cca02251 <FixedArray[0]> {}
 - embedder fields = {
    (nil)
    (nil)
 }
pwndbg> x/40gx 0x2ca728c8d469-1
	0x2ca728c8d468:	0x00000796fdd83d31	0x00002bf2cca02251
	0x2ca728c8d478:	0x00002bf2cca02251	0x0000123400000000
	0x2ca728c8d488:	0x000055eba87e2650	0x000055eba87e2650
	0x2ca728c8d498:	0x0000000000001234	0x0000000000000004
	0x2ca728c8d4a8:	0x0000000000000000	0x0000000000000000
	0x2ca728c8d4b8:	0x000010772d302751	0x0000000077a69296
	0x2ca728c8d4c8:	0x000006b300000000	0x6974636e7566280a
```

backing_store -> buffer

```
pwndbg> x/40gx 0x55eba87e2650
0x55eba87e2650:	0x5566778811223344	0x0000000000000000
0x55eba87e2660:	0x0000000000000000	0x0000000000000000
0x55eba87e2670:	0x0000000000000000	0x0000000000000000
0x55eba87e2680:	0x0000000000000000	0x0000000000000000
```

map

```
pwndbg> job 0x796fdd83d31
0x796fdd83d31: [Map]
 - type: JS_ARRAY_BUFFER_TYPE
 - instance size: 80
 - inobject properties: 0
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - back pointer: 0x2bf2cca022e1 <undefined>
 - instance descriptors (own) #0: 0x2bf2cca02231 <DescriptorArray[2]>
 - layout descriptor: (nil)
 - prototype: 0x1f0ae892ac9 <Object map = 0x796fdd83d81>
 - constructor: 0x1f0ae892929 <JSFunction ArrayBuffer (sfi = 0x2bf2cca24299)>
 - dependent code: 0x2bf2cca02251 <FixedArray[0]>
 - construction counter: 0
pwndbg> x/40gx 0x796fdd83d31-1
	0x796fdd83d30:	0x000010772d302251	0x0d000423110a0a0a
	0x796fdd83d40:	0x00000000082003ff	0x000001f0ae892ac9
	0x796fdd83d50:	0x000001f0ae892929	0x0000000000000000
	0x796fdd83d60:	0x00002bf2cca02231	0x0000000000000000
	0x796fdd83d70:	0x00002bf2cca02251	0x0000000000000000
```

### Unboxed Array

e.g.  `FixedDoubleArray` 

```
[1,2,3,4]
[1.1, 2.2,3.3,4.4]
```



### Boxed Array

code

```javascript
console.log([1.1, 0x11223344 , new ArrayBuffer(0x1234)]) 
```

double을 꼭 넣어야 한다. int만 넣게된다면 object가 아닌 primitive한 int가 저장되게 된다.



object

```
pwndbg> job *args.values_
0x1de7db1905e9: [JSArray]
 - map: 0x2a18050826b1 <Map(PACKED_ELEMENTS)> [FastProperties]
 - prototype: 0x318f3a85691 <JSArray[0]>
 - elements: 0x1de7db1906d9 <FixedArray[3]> [PACKED_ELEMENTS]
 - length: 3
 - properties: 0x3369bec82251 <FixedArray[0]> {
    #length: 0x3369becad459 <AccessorInfo> (const accessor descriptor)
 }
 - elements: 0x1de7db1906d9 <FixedArray[3]> {
           0: 0x1de7db190701 <Number 4.58242e+09>
           1: 0x1de7db190711 <Number 1.1>
           2: 0x1de7db190641 <ArrayBuffer map = 0x2a1805083d31>
 }
pwndbg> x/40gx 0x1de7db1905e9-1
0x1de7db1905e8:	0x00002a18050826b1	0x00003369bec82251
0x1de7db1905f8:	0x00001de7db1906d9	0x0000000300000000
0x1de7db190608:	0x00001adde1c84321	0x00000318f3aaba59
0x1de7db190618:	0x00001adde1c82f71	0x0000000300000000
0x1de7db190628:	0x41f1122334400000	0x3ff199999999999a
0x1de7db190638:	0x0000000000000000	0x00002a1805083d31
```

elements

```

pwndbg> job 0x1de7db1906d9
0x1de7db1906d9: [FixedArray]
 - map: 0x1adde1c82341 <Map(HOLEY_ELEMENTS)>
 - length: 3
           0: 0x1de7db190701 <Number 4.58242e+09> //double이 아닌 int를 넣게되면 0x11223344가 그대로 들어감
           1: 0x1de7db190711 <Number 1.1>
           2: 0x1de7db190641 <ArrayBuffer map = 0x2a1805083d31>
pwndbg> x/40gx 0x1de7db1906d9-1
0x1de7db1906d8:	0x00001adde1c82341	0x0000000300000000
0x1de7db1906e8:	0x00001de7db190701	0x00001de7db190711
0x1de7db1906f8:	0x00001de7db190641	0x00001adde1c82521
0x1de7db190708:	0x41f1122334400000	0x00001adde1c82521 //0x11223344
0x1de7db190718:	0x3ff199999999999a	0xdeadbeedbeadbeef //1.1
```

Elements.map

```
pwndbg> job 0x1adde1c82341
0x1adde1c82341: [Map]
 - type: FIXED_ARRAY_TYPE
 - instance size: 0
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - non-extensible
 - back pointer: 0x3369bec822e1 <undefined>
 - instance descriptors (own) #0: 0x3369bec82231 <DescriptorArray[2]>
 - layout descriptor: (nil)
 - prototype: 0x3369bec82201 <null>
 - constructor: 0x3369bec82201 <null>
 - dependent code: 0x3369bec82251 <FixedArray[0]>
 - construction counter: 0
```

#### value access

code

```javascript
arr = [1.1, {}]
%DebugPrint(arr)
arr[0] = 2.2
%DebugPrint(arr)
```



```
 - elements: 0x2caebf490541 <FixedArray[2]> {
           0: 0xec0a57abcb1 <Number 1.1>
           1: 0x2caebf490561 <Object map = 0x1407bb002391>
 }
- elements: 0x2caebf490541 <FixedArray[2]> {
           0: 0xec0a57ac2e9 <Number 2.2>
           1: 0x2caebf490561 <Object map = 0x1407bb002391>
 }
```

### Function

code

```javascript
function foo(a,b) { return a + b; }
%DebugPrint(foo);
```

object

```
pwndbg> job 0x2c2c283ab2f9
0x2c2c283ab2f9: [Function] in OldSpace
 - map: 0x1514f20824d1 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x2c2c28384759 <JSFunction (sfi = 0x1d7204505521)>
 - elements: 0x1d7204502251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - function prototype:
 - initial_map:
 - shared_info: 0x2c2c283ab109 <SharedFunctionInfo foo>
 - name: 0x2c2c283aae11 <String[3]: foo>
 - builtin: CompileLazy
 - formal_parameter_count: 2
 - kind: NormalFunction
 - context: 0x2c2c28383eb1 <FixedArray[275]>
 - code: 0x437a8999921 <Code BUILTIN>
 - source code: (a,b) { return a + b; }
 - properties: 0x1d7204502251 <FixedArray[0]> {
    #length: 0x1d720452d769 <AccessorInfo> (const accessor descriptor)
    #name: 0x1d720452d6f9 <AccessorInfo> (const accessor descriptor)
    #arguments: 0x1d720452d619 <AccessorInfo> (const accessor descriptor)
    #caller: 0x1d720452d689 <AccessorInfo> (const accessor descriptor)
    #prototype: 0x1d720452d7d9 <AccessorInfo> (const accessor descriptor)
 }

 - feedback vector: not available
 
 pwndbg> x/40gx 0x2c2c283ab2f9-1
0x2c2c283ab2f8:	0x00001514f20824d1	0x00001d7204502251
0x2c2c283ab308:	0x00001d7204502251	0x00002c2c283ab109
0x2c2c283ab318:	0x00002c2c28383eb1	0x00002c2c283ab2d9
0x2c2c283ab328:	0x00000437a8999921	0x00001d7204502321 code
0x2c2c283ab338:	0x000001420aa02981	0x0000966000000000
0x2c2c283ab348:	0x00002c2c283aae11	0x00002c2c283ab2f9
```

code

```
pwndbg> job 0x437a8999921
0x437a8999921: [Code]
 - map: 0x1420aa02841 <Map(HOLEY_ELEMENTS)>
kind = BUILTIN
name = CompileLazy
compiler = unknown
address = 0x437a8999921
Body (size = 1212)
Instructions (size = 1212)
0x437a8999980     0  488b5f27       REX.W movq rbx,[rdi+0x27]
0x437a8999984     4  488b5b07       REX.W movq rbx,[rbx+0x7]
0x437a8999988     8  493b5da0       REX.W cmpq rbx,[r13-0x60]
0x437a899998c     c  0f8431040000   jz 0x437a8999dc3  (CompileLazy)
0x437a8999992    12  488b4b0f       REX.W movq rcx,[rbx+0xf]
0x437a8999996    16  f6c101         testb rcx,0x1
0x437a8999999    19  0f852e020000   jnz 0x437a8999bcd  (CompileLazy)
0x437a899999f    1f  f6c101         testb rcx,0x1
0x437a89999a2    22  7410           jz 0x437a89999b4  (CompileLazy)
0x437a89999a4    24  48ba000000001f000000 REX.W movq rdx,0x1f00000000
0x437a89999ae    2e  e86dc20100     call 0x437a89b5c20  (Abort)    ;; code: BUILTIN
...
RelocInfo (size = 52)
0x437a89999af  code target (BUILTIN Abort)  (0x437a89b5c20)
0x437a89999d7  code target (BUILTIN Abort)  (0x437a89b5c20)
0x437a89999e9  embedded object  (0x437a8999921 <Code BUILTIN>)
```

code + 0x60에 **JIT code**가 있다.

## Constants

### instance type (map type)

JS_ARRAY_TYPE = 0x0424

```
JS_API_OBJECT_TYPE = 0x0420,
JS_OBJECT_TYPE,
JS_ARGUMENTS_TYPE,
JS_ARRAY_BUFFER_TYPE, //0x0423
JS_ARRAY_TYPE, //0x0424
```

FIXED_ARRAY_TYPE = 0x00b2

```
// FixedArrays.
FIXED_ARRAY_TYPE,  // FIRST_FIXED_ARRAY_TYPE // 0x00b2
DESCRIPTOR_ARRAY_TYPE,
HASH_TABLE_TYPE,
SCOPE_INFO_TYPE,
TRANSITION_ARRAY_TYPE,  // LAST_FIXED_ARRAY_TYPE
```

FIXED_DOUBLE_ARRAY_TYPE = 0x0096

```
FIXED_BIGUINT64_ARRAY_TYPE,  // LAST_FIXED_TYPED_ARRAY_TYPE
FIXED_DOUBLE_ARRAY_TYPE,
FILLER_TYPE,  // LAST_DATA_TYPE
```

HOLEY_ELEMENTS = 0x00b2

```
PACKED_ELEMENTS,
HOLEY_ELEMENTS,

// The "fast" kind for unwrapped, non-tagged double values.
PACKED_DOUBLE_ELEMENTS,
HOLEY_DOUBLE_ELEMENTS,

```

JS_DATA_VIEW_TYPE = 0x0439



### elements kind(ElementsKind)

HOLEY_ELEMENTS = 0x0d00

PACKED_SMI_ELEMENTS = 0x100

PACKED_DOUBLE_ELEMENTS = 0x1100

HOLEY_DOUBLE_ELEMENTS = 0x1500



## Exploit

* boxed array와 unboxed array를 적절히 활용.

* low한 값(address,double, integer 등)을 access할 때는 unboxed array를 이용
  * Example
  ```javascript
  boxed[IDX] = obj; 
  console.log(unboxed[IDX]);
  ```

* object를 가져올 때에는 boxed array 이용
  * Example
  ```javascript
  unboxed[IDX] = u2d(fakeDataViewAddr); 
  DataView.prototype.getUint32(oobBoxed[IDX], 0, true);
  ```

* leak한 fake map은 double array 로 구성했기 때문에 정확히는 object의 address이다.  fake map address를 정확히 알기 위해선 `elements`멤버의 address를 알아야 하므로 leak을 한후 elements의 offset을 알아야내야한다. 

  ```
  0x11e14ab0d4c1: [JSArray]
   - map: 0x34c3f5382611 <Map(PACKED_DOUBLE_ELEMENTS)> [FastProperties]
   - prototype: 0x1493f3e85691 <JSArray[0]>
   - elements: 0x11e14ab0d4f1 <FixedDoubleArray[2]> [PACKED_DOUBLE_ELEMENTS]
   - length: 2
   - properties: 0x319a2ee82251 <FixedArray[0]> {
      #length: 0x319a2eead459 <AccessorInfo> (const accessor descriptor)
   }
   - elements: 0x11e14ab0d4f1 <FixedDoubleArray[2]> {
   
  pwndbg> x/40gx 0x11e14ab0d4f1-1
  	0x11e14ab0d4f0:	0x00003235f3c02f71	0x0000000200000000 map | length
  	0x11e14ab0d500:	0x3ff199999999999a	0x43b1223344556678 1.1 | 
  	0x11e14ab0d510:	0xdeadbeedbeadbeef	0xdeadbeedbeadbeef
  ```
  * Object(JSArray) Address : 0x11e14ab0d4c1
  * element address : 0x11e14ab0d4f0
  * element buffer address : 0x11e14ab0d500
  * element buffer address - Object address = 0x3f
  * fake map address element buffer address offset = 0x3f

* elements offset을 알아낼때 `.slice()`를 사용해야한다. 

  * `.slice()`는 object를 copy한다. 

  * 이를 사용하지 않을경우 elements offset이 간헐적으로 바뀌어 맞지 않는 경우가 발생한다.

  * 위에선 일반적인 elements offset을 알아내는법을 썼지만 `.slice()`를 사용할경우

    ```
    DebugPrint: 0x105f9c7126e9: [JSArray]
     - elements: 0x105f9c7126a9 <FixedDoubleArray[6]> {
    
    ```

    object보다 elements 더 낮은 주소로 할당되게 된다. 그렇기 떄문에 offset을 -해야한다.

## References

### CTF
[jsExploit_CTF/hard_exploit.js at master · uknowy/jsExploit_CTF · GitHub](https://github.com/uknowy/jsExploit_CTF/blob/master/WithCon2016/hard_exploit.js)
[GitHub - l0kihardt/jsExploit_CTF: JavaScript Engine Exploits in CTF](https://github.com/l0kihardt/jsExploit_CTF)

### 1 day exploit

[Exploiting the Math.expm1 typing bug in V8 | 0x41414141 in ?? ()](https://abiondo.me/2019/01/02/exploiting-math-expm1-v8/)
[SSD Advisory – Chrome Type Confusion in JSCreateObject Operation to RCE – SecuriTeam Blogs](https://blogs.securiteam.com/index.php/archives/3783)

### V8 Engine Internal
[자바스크립트는 어떻게 작동하는가: V8 엔진의 내부 + 최적화된 코드를 작성을 위한 다섯 가지 팁](https://engineering.huiseoul.com/%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EC%9E%91%EB%8F%99%ED%95%98%EB%8A%94%EA%B0%80-v8-%EC%97%94%EC%A7%84%EC%9D%98-%EB%82%B4%EB%B6%80-%EC%B5%9C%EC%A0%81%ED%99%94%EB%90%9C-%EC%BD%94%EB%93%9C%EB%A5%BC-%EC%9E%91%EC%84%B1%EC%9D%84-%EC%9C%84%ED%95%9C-%EB%8B%A4%EC%84%AF-%EA%B0%80%EC%A7%80-%ED%8C%81-6c6f9832c1d9)
[How JavaScript works: inside the V8 engine + 5 tips on how to write optimized code](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)
[GitHub - danbev/learning-v8: Project for learning V8 internals](https://github.com/danbev/learning-v8)
[Documentation · V8](https://v8.dev/docs)
[Fast properties in V8 · V8](https://v8.dev/blog/fast-properties)

### Turbo Exploit
[SSD Advisory – Chrome Turbofan Remote Code Execution – SecuriTeam Blogs](https://blogs.securiteam.com/index.php/archives/3379)
[Fast properties in V8 · V8](https://v8.dev/blog/fast-properties)

### JIT mechanism & exploit
[Source to Binary - journey of V8 javascript engine (English version) - Speaker Deck](https://speakerdeck.com/brn/source-to-binary-journey-of-v8-javascript-engine-english-version)
[Attacking Client-Side JIT Compilers - Samuel PDF](https://saelo.github.io/presentations/blackhat_us_18_attacking_client_side_jit_compilers.pdf)

### V8 Exploit in Chinese & Japanese

[v8 exploit](http://eternalsakura13.com/2018/05/06/v8/)

### Optimizer

[An Introduction to Speculative Optimization in V8 · Benedikt Meurer](https://benediktmeurer.de/2017/12/13/an-introduction-to-speculative-optimization-in-v8/)

### Blog
[Blog · Benedikt Meurer](https://benediktmeurer.de/)
[标签: v8 | Sakuraのblog](http://eternalsakura13.com/tags/v8/page/2/) 
