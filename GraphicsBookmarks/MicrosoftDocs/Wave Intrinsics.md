## **Introduction:**

Up until now, the HLSL programming model has exposed only a single thread of execution. As of v6.0, new wave-level operations are provided to explicitly take advantage of the fact that on current GPUs, many threads can be executing in lockstep on the same core simultaneously. For example, these intrinsics enable the elimination of barrier constructs when the scope of synchronization is within the width of the SIMD processor. These can also work in the case of some other grouping of threads that are known to be atomic relative to each other.

Potential use cases include performance improvements to: stream compaction, reductions, block transpose, bitonic sort or FFT, binning, stream de-duplication, etc.

All non-quad related Wave Intrinsics are available in all shader stages. Quad wave intrinsics are available only in pixel and compute shaders.

These intrinsics operate as though the following statement was performed by default:

```
@import waveOps.h;
```

This statement is not supported or required in shader model 6.0 shaders. It will be available once there is support for import libraries / modules.

## [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#glossary)**Glossary:**

Lane:     A single thread of execution. The shader models before version 6.0 expose only one of these at the language level, leaving distribution across parallel SIMD processors entirely up to the implementation.

Wave:   A set of lanes executed simultaneously in the processor. No explicit barriers are required to guarantee that they execute in parallel. Similar concepts include “warp” and “wavefront”.

Helper Lane: A lane which is executed solely for the purpose of gradients in pixel shader quads. The output of such a lane will be discarded, and so not rendered to the destination surface. UAV accesses and most wave intrinsics are disabled on helper lanes.

Inactive Lane: a lane which is not being executed at this point in the code, e.g. due to flow control, or insufficient work to fill this lane in the wave.

Active Lane: A lane for which execution is currently being performed as determined by flow control and initial launch conditions.

Quad: A set of 4 adjacent lanes corresponding to pixels arranged in a 2x2 square. They are used to estimate gradients by differencing in either x or y. A wave may be comprised of multiple quads.

> All pixels in an active quad are potentially executed (may be active at some point). Those that do not produce visible results are termed helper lanes.

## [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#caps-flags)**Caps Flags:**

`BOOL WaveOps`: The driver should expose the waveOps caps flag if it can support the intrinsics in this specification. The driver must set this cap for the D3D runtime to load shaders containing these intrinsics. On implementations that do not set this bit, CreateShader() will fail on such shaders.

`UINT WaveLaneCountMin`: this cap specifies the baseline number of lanes in the SIMD wave that this implementation can support. This term is sometimes known as “wavefront size” or “warp width”. This capability value is exposed at the API level. Currently apps should rely on only this minimum value for sizing workloads.

`UINT WaveLaneCountMax`: this cap specifies the maximum number of lanes in the SIMD wave that this implementation can support. (aka maximum “wavefront size” or “warp width”). This cap is reserved for future expansion, and is not expected to be used by applications initially.

## [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#operation)**Operation:**

These intrinsics are dependent on active lanes and therefore flow control. In the model of this document, implementations must enforce that the number of active lanes exactly corresponds to the programmer’s view of flow control. In a future version, there may be a compiler flag to relax this requirement as a default, but also enable applications to be explicit about the exact set of lanes to be used in a particular wave operation (see section Wave Handles in the Future Features section below).

## [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#direct3d-api-additions)**Direct3D API Additions:**

The only Direct3D API change is that the above Capabilities flags (shader model 6 and wave intrinsics) are made visible to applications via the API. All the intrinsics appear only in HLSL.

## [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#shading-language-intrinsics)**Shading Language Intrinsics:**

The following new intrinsics are added to HLSL for use in shader model 6 and higher. The term “current wave” refers to the wave of lanes in which the program is executing.

All wave operations with the exception of [Wave Query Intrinsics](https://github.com/Microsoft/DirectXShaderCompiler/wiki/wave-query-intrinsics) and [Quad-Wide Shuffle Operations](https://github.com/Microsoft/DirectXShaderCompiler/wiki/quad-wide-shuffle-operations) are disabled on helper lanes. They are treated as if flow control excludes these operations on helper lanes, therefore values read from or returned to helper lanes by these operations are undefined.

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#wave-query-intrinsics)Wave Query Intrinsics:

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#bool-waveisfirstlane)`bool WaveIsFirstLane()`

This result returns true only for the active, non-helper lane in the current wave with the smallest index. It can be used to identify operations that are to be executed only once per wave.

Example:

```
if ( WaveIsFirstLane() )
{
    . . . // once per-wave code
}
```

> Caution must be used in the presence of helper lanes, since false will always be returned on helper lanes.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#uint-wavegetlanecount)`uint WaveGetLaneCount()`

Returns the number of lanes in a wave on this architecture. The result will be between 4 and 128. Includes all lanes (active, inactive and/or helper lanes).

Example:

```
uint laneCount = WaveGetLaneCount(); // number of lanes in wave
```

_Note: the result returned from this routine may vary significantly depending on the implementation (vendor, generation, architecture, etc.)._

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#uint-wavegetlaneindex)`uint WaveGetLaneIndex()`

Returns the index of the current lane within the current wave. The result must be in the [0, WaveGetLaneCount) range.

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#wave-vote-intrinsics)Wave Vote Intrinsics:

This set of intrinsics compare values across threads currently active from the current wave.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#bool-waveactiveanytrue-bool-expr-)`bool WaveActiveAnyTrue( bool expr )`

Returns true if <expr> is true in any active non-helper lane in the current wave.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#bool-waveactivealltrue-bool-expr-)`bool WaveActiveAllTrue( bool expr )`

Returns true if <expr> is true in all active non-helper lanes in the current wave.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#uint4-waveactiveballot-bool-expr-)`uint4 WaveActiveBallot( bool expr )`

Returns a uint4 containing a bitmask of the evaluation of the Boolean <expr> for all active non-helper lanes in the current wave. The least-significant bit corresponds to the lane with index zero. The bits corresponding to inactive or helper lanes will be zero. The bits that are greater than or equal to WaveGetLaneCount will be zero.

Example:

```
// get a bitwise representation of the number of currently active non-helper lanes:
uint4 waveBits = WaveBallot( true ); // convert to bits
```

_Note: the number of bits set in the result of this routine may vary significantly depending on the implementation (vendor, generation, architecture, etc.), so should not be relied on for portable code_

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#types)Types:

The expression `<type>` implies the type of the expression, the supported types are those from the following list that are also present in the target shader model for the program:

> half, half2, half3, half4 float, float2, float3, float4 double, double2, double3, double4 int, int2, int3, int4 uint, uint2, uint3, uint4 short, short2, short3, short4 ushort, ushort2, ushort3, ushort4 uint64_t, uint64_t2, uint64_t3, uint64_t4

Note: some operations (bitwise operators) only support the integer types.

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#wave-broadcast-or-shuffle-intrinsics)Wave Broadcast or Shuffle Intrinsics

The following routines enable all active non-helper lanes in the current wave to receive the value(s) from the specified lane(s). The return value from an inactive lane or helper lane is undefined.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-wavereadlanefirst-type-expr-)`<type> WaveReadLaneFirst( <type> expr )`

Returns the value of expr for the active non-helper lane of the current wave with the smallest index. The resulting value is thus uniform across the wave, making this effectively a broadcast operation.

> Caution must be exercised in the presence of helper lanes, since common patterns using this intrinsic can produce unexpected results or undefined behavior.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-wavereadlaneat-type-expr-uint-laneindex)`<type> WaveReadLaneAt( <type> expr, uint laneIndex)`

Returns the value of expr for the given lane index within the current wave.

If laneIndex is uniform, this is effectively a broadcast operation, otherwise it is a shuffle operation.

The result is undefined on a helper lane, or if the lane referred to by laneIndex is inactive or a helper lane.

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#wave-reduction-intrinsics)Wave Reduction Intrinsics

These intrinsics compute the specified operation across all active non-helper lanes in the wave and broadcast the final result to all active non-helper lanes. Therefore, the final output is guaranteed uniform across the wave.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#bool-waveactiveallequaltype-expr-)`bool WaveActiveAllEqual(<type> expr )`

Returns true if <expr> is the same for every active non-helper lane in the current wave (and thus uniform across it).

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#uint-waveactivecountbits-bool-bbit-)`uint WaveActiveCountBits( bool bBit )`

Counts the number of Boolean variables (bBit) which evaluate to true across all active non-helper lanes in the current wave, and replicates the result to all lanes in the wave. Providing an explicit `true` Boolean value returns the number of active non-helper lanes.

This can be implemented more efficiently than a full `WaveActiveSum()` via something like:

```
result = countbits( waveBallot( bBit ) );
```

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-waveactivesum-type-expr-)`<type> WaveActiveSum( <type> expr )`

Sums up the value of <expr> across all active non-helper lanes in the current wave, and replicates it to all lanes in said wave. The order of operations is undefined.

Example:

```
float3 total = WaveActiveSum( position ); // sum positions in wave
float3 center = total/count; // compute average of these positions
```

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-waveactiveproduct-type-expr)`<type> WaveActiveProduct( <type> expr)`

Multiplies the values of <expr> together across all active non-helper lanes in the current wave and replicates it back to all active non-helper lanes. The order of operations is undefined.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#int_type-waveactivebitand-int_type-expr)`<int_type> WaveActiveBitAnd( <int_type> expr)`

Returns the bitwise AND of all the values of <expr> across all active non-helper lanes in the current wave and replicates it back to all active non-helper lanes.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#int_type-waveactivebitor-int_type-expr-)`<int_type> WaveActiveBitOr( <int_type> expr )`

Returns the bitwise OR of all the values of <expr> across all active non-helper lanes in the current wave and replicates it back to all active non-helper lanes.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#int_type-waveactivebitxor-int_type-expr)`<int_type> WaveActiveBitXor( <int_type> expr)`

Returns the bitwise Exclusive OR of all the values of <expr> across all active non-helper lanes in the current wave and replicates it back to all active non-helper lanes.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-waveactivemin-type-expr)`<type> WaveActiveMin( <type> expr)`

Computes minimum value of <expr> across all active non-helper lanes in the current wave and replicates it back to all active non-helper lanes. The order of operations is undefined.

Example:

```
float3 minPos = WaveActiveMin( myPoint.position );
BoundingBox.min = min( minPos, BoundingBox.min );
```

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-waveactivemax-type-expr)`<type> WaveActiveMax( <type> expr);`

Computes maximum value of <expr> across all active non-helper lanes in the current wave and replicates it back to all active non-helper lanes. The order of operations is undefined.

Example:

```
float3 maxPos = WaveActiveMax( myPoint.position );
BoundingBox.max = max( maxPos, BoundingBox.max );
```

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#wave-scanprefix-intrinsics)Wave Scan/Prefix Intrinsics

The following intrinsics apply the operation to each lane and leave each partial result of the computation in the corresponding lane, e.g. wavePrefixSum() returns for each lane the sum of elements up to but not including that lane. Note: Postfix versions of the Prefix routines can be implemented by adding in the current lane’s value.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#uint-waveprefixcountbits-bool-bbit-)`uint WavePrefixCountBits( Bool bBit )`

Returns the sum of all the specified Boolean variables (bBit) set to true across all active non-helper lanes with indices smaller than this lane’s. A postfix version is implemented by adding the current lane’s bit value.

> The active non-helper lane with the lowest index will always receive a 0.

This can be implemented more efficiently than a full WavePrefixSum() via the following pseudo code:

```
uint bits = WaveBallot( bBit );
laneMaskLT = (1 << WaveGetLaneIndex()) - 1;
prefixBitCount = countbits( bits & laneMaskLT);
```

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#example)Example:

Use `WavePrefixCountBits()` to implement a compacted write to an ordered stream where the number of elements written per lane is either 1 or 0.

```
bool bDoesThisLaneHaveAnAppendItem = <expr>;
// compute number of items to append for the whole wave
uint laneAppendOffset = WavePrefixCountBits( bDoesThisLaneHaveAnAppendItem );
uint appendCount = WaveActiveCountBits( bDoesThisLaneHaveAnAppendItem);
// update the output location for this whole wave
uint appendOffset;
if ( WaveIsFirstLane () )
{
    // this way, we only issue one atomic for the entire wave, which reduces contention
    // and keeps the output data for each lane in this wave together in the output buffer
    appendOffset = atomicAdd(bufferSize, appendCount);
}
appendOffset = WaveReadFirstLane( appendOffset ); // broadcast value
appendOffset += laneAppendOffset; // and add in the offset for this lane
buffer[appendOffset] = myData; // write to the offset location for this lane
```

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-waveprefixproduct-type-value-)`<type> WavePrefixProduct( <type> value )`

Returns the product of all of the s in the active non-helper lanes in this wave with indices less than this lane.

The order of operations on this routine cannot be guaranteed, so effectively the [precise] flag is ignored within it. A postfix product can be computed by multiplying the prefix product by the current lane’s value.

> The active non-helper lane with the lowest index will always receive a 1.

Example:

```
uint numToMultiply = 2;
uint prefixProduct = WavePrefixProduct( numToMultiply );
```

On a machine with a wave size of 8 and all lanes active and not helpers except lanes 0 and 4 the following values would be returned from WavePrefixProduct.

lane index

status

prefixProduct

0

inactive

n/a

1

active

= 1

2

active

= 1*2

3

active

= 1*2*2

4

inactive

n/a

5

active

= 1*2*2*2

6

active

= 1*2*2*2*2

7

active

= 1*2*2*2*2*2

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-waveprefixsum-type-value-)`<type> WavePrefixSum( <type> value )`

Returns the sum of all of the <value>s in the active non-helper lanes of this wave having indices less than this one.

The order of operations on this routine cannot be guaranteed, so effectively the [precise] flag is ignored within it. A postfix sum can be computed by adding the prefix sum to the current lane’s value.

> The active non-helper lane with the lowest index will always receive a 0.

Example:

```
uint numToSum = 2;
uint prefixSum = WavePrefixSum( numToSum );
```

On a machine with a wave size of 8 and all lanes active and not helpers except lanes 0 and 4 the following values would be returned from WavePrefixSum.

lane index

status

prefixProduct

0

inactive

n/a

1

active

= 0

2

active

= 0+2

3

active

= 0+2+2

4

inactive

n/a

5

active

= 0+2+2+2

6

active

= 0+2+2+2+2

7

active

= 0+2+2+2+2+2

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#example-ordered-append)Example: Ordered Append

This code demonstrates use of the above intrinsics to implement a compacted write to an ordered stream where the number of elements written varies per lane.

```
bool doesThisLaneHaveAnAppendItem = (NumberOfItemsToAppend) ? 1 : 0;

// compute number of items to append for the whole wave
uint appendCount = WaveActiveCountBits( doesThisLaneHaveAnAppendItem );

// compute number of locations filled before this one
uint laneAppendOffset = WavePrefixSum( NumberOfItemsToAppend );

// update the output location for this whole wave
uint appendOffset;
if ( WaveIsFirstLane() ) // execute only once per wave
{
    // this way, we only issue one atomic for the entire wave, which reduces contention
    // and keeps the output data for each lane in this wave together in the output buffer

    appendOffset = atomicAdd(bufferSize, appendCount);
}
appendOffset = WaveReadFirstLane(appendOffset); // broadcast value
appendOffset += laneAppendOffset; // and add in the offset for this lane
buffer[appendOffset] = myData; // write to the offset location for this lane
```

### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#quad-wide-shuffle-operations)Quad-Wide Shuffle Operations

These intrinsics perform swap operations on the values across a wave known to contain pixel shader quads as defined here. The indices of the pixels in the quad are defined in scan-line or reading order:

-   +---------> X
-   | [0] [1]
-   | [2] [3]
-   v
-   Y

Where the coordinates are [0] is at x,y, [1] is at [x+1,y], [2] is at [x, y+1], and [3] is at [x+1, y+1].

These routines work in either compute shaders or pixel shaders. In compute shaders they operate in quads defined as evenly divided groups of 4 within a SIMD wave, for example _quadID = WaveGetLaneIndex() / 4; quadIndex = WaveGetLaneIndex() % 4_

Since these routines rely on quad-level values, they assume that all lanes in the quad are active, including helper lanes (those that are masked from final writes). This means they should be treated like the existing DDX and DDY intrinsics in that sense.

Unlike for most other wave intrinsics, for these routines, reading from active helper lanes is well defined, and the return value is also well-defined on helper lanes.

These routines assume that flow control execution is uniform at least across the quad.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-quadreadacrossx-type-localvalue-)`<type> QuadReadAcrossX( <type> localValue )`

Returns the specified local value read from the other lane in this quad in the X direction.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-quadreadacrossy-type-localvalue-)`<type> QuadReadAcrossY( <type> localValue )`

Returns the specified local value read from the other lane in this quad in the Y direction.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-quadreadacrossdiagonal-type-localvalue-)`<type> QuadReadAcrossDiagonal( <type> localValue )`

Returns the specified local value which is read from the diagonally opposite lane in this quad.

#### [](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics#type-quadreadlaneat-type-sourcevalue-uint-quadlaneid-)`<type> QuadReadLaneAt( <type> sourceValue, uint quadLaneID )`

Returns the specified source value from the lane identified by quadLaneID within the current quad.

`quadLaneID` is not required to be uniform, meaning this can be considered a broadcast if `quadLaneID` is uniform, or a shuffle if not.