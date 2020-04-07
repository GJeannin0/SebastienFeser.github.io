# Quaternions Intrinsics Optimisation
I’ve been working on the optimisation of Quaternions for the Neko Game Engine we’re currently working on.

Quaternions are used to represent rotations. They’re based on complex numbers and aren’t easy to understand intuitively. You almost never access or modify individual Quaternion components. You’d just take existing rotation like the one in the transform and use them to construct new rotations. One of the main reason we’re using Quaternions is to avoid gimbal lock which is the loss of one degree of freedom in a rotation.

**3 Axes Gimbal Rotation**

![](https://upload.wikimedia.org/wikipedia/commons/5/5a/Gimbal_3_axes_rotation.gif)

**Gimball Lock**

![](https://upload.wikimedia.org/wikipedia/commons/4/49/Gimbal_Lock_Plane.gif)


<!--- Show a picture illustrating quaternions and rotations --->

My task was to optimise their use as much as I could so I decided to optimise them by working with the Intel Intrinsics functions.

Let’s have a look

## Quaternion
The basic implementation of the Quaternions looks like this:

```cpp
struct Quaternion
{
  float x;        //4 bytes
  float y;        //4 bytes
  float z;        //4 bytes
  float w;        //4 bytes
};
```

<!--- Show in code the implementation --->

The *Quaternion* contains 4 *floats* representing every values.

For the optimization I implemented, I decided to take the **Dot** function that calculates the Dot Product of 2 *Quaternions*, as a reference to test if it was working or not.

So here’s how the **Dot** function of my *Quaternion* struct looks like:
```cpp
static float Dot(Quaternion a, Quaternion b)
{
  return  a.x * b.x +
          a.y * b.y +
          a.z * b.z +
          a.w * b.w;
}
```
This is how the x64 msvc v19.24 compiler compiles the Dot product code in assembly:
```Assembly
float Dot(Quaternion,Quaternion) PROC                      ; Dot
        mov     QWORD PTR [rsp+16], rdx
        mov     QWORD PTR [rsp+8], rcx
        mov     rax, QWORD PTR a$[rsp]
        mov     rcx, QWORD PTR b$[rsp]
        movss   xmm0, DWORD PTR [rax]
        mulss   xmm0, DWORD PTR [rcx]
        mov     rax, QWORD PTR a$[rsp]
        mov     rcx, QWORD PTR b$[rsp]
        movss   xmm1, DWORD PTR [rax+4]
        mulss   xmm1, DWORD PTR [rcx+4]
        addss   xmm0, xmm1
        mov     rax, QWORD PTR a$[rsp]
        mov     rcx, QWORD PTR b$[rsp]
        movss   xmm1, DWORD PTR [rax+8]
        mulss   xmm1, DWORD PTR [rcx+8]
        addss   xmm0, xmm1
        mov     rax, QWORD PTR a$[rsp]
        mov     rcx, QWORD PTR b$[rsp]
        movss   xmm1, DWORD PTR [rax+12]
        mulss   xmm1, DWORD PTR [rcx+12]
        addss   xmm0, xmm1
        ret     0
float Dot(Quaternion,Quaternion) ENDP                      ; Dot
```
It first adds each values of the Quaternions, then multiply a.x and b.x and after a.y and b.y and then add them. After that he does the exact same with a.z, b.z, a.w and b.w. Finally he add the two values together so we have our Dot product result.

The **Dot** function takes each value of the *Quaternion a* and the *Quaternion b* to calculate the Dot product and it returns a *float*.

These are the most logical way someone would implement a Quaternion and a Dot Product. Now, let’s look how I decided to optimise this calculus.

## FourQuaternion
To optimise my code, I decided to create a new Struct called *FourQuaternion*. The idea here is that this structure will contain 4 *Quaternions* instead of one.

Why am I doing that? Because instead of doing the calculus 4 times with a different *Quaternion*, I’ll be doing it only once by aligning each values of the Quaternions.

The *FourQuaternion* structure looks like this:

```cpp
struct FourQuaternion
{
  std::array<float, 4> x;       //16 bytes
  std::array<float, 4> y;       //16 bytes
  std::array<float, 4> z;       //16 bytes
  std::array<float, 4> w;       //16 bytes
};
```
<!--- Show in code the implementation --->

It contains an array of 4 *floats* for each values in the *FourQuaternion*.

## Array of Structures of Arrays
I've decided to approach the problem by creating a AoSoA system.

To stock the values of my *FourQuaternions*, I decided to use Array of Structure of arrays.
<!--- explain more about AoS SoA and AoSoA --->
Structure of arrays is a layout separating elements of a structure into one parallel array per field. It makes it easier to use them by packing them into SIMD instructions. So it’s necessary I create those first.

![](ScalarVsSIMD.png)

The reason why structures of arrays are better here is because the values will be aligned in code so it's much faster to load all values from memory in one block instead of going for each values and then aligning them.

- **Aos Alignement:** xyzwxyzwxyzwxyzw
- **SoA Alignment:** xxxxyyyyzzzzwwww

Every FourQuaternions will be stocked in an Array in the code, this is then an Array of Structures.

So combining these two values, it creates an Array of Structures of Array.

This is how I decided to implement my AoSoA:

![](FourQuaternionsAoSoA.png)

## Intel Intrinsics
To do the functions with these array, I’ll have to use the Intel intrinsic instructions, which are C style functions that provide access to many Intel instructions without the need to write assembly code. 

Since I decided to use the **Dot** function as a test, let’s look how it looks like in the *FourQuaternion*:
```cpp
static inline std::array<float, 4> Dot(FourQuaternion q1, FourQuaternion q2)
	{
		alignas(4 * sizeof(float)) std::array<float, 4> result;
		auto x1 = _mm_load_ps(q1.x.data());
		auto y1 = _mm_load_ps(q1.y.data());
		auto z1 = _mm_load_ps(q1.z.data());
		auto w1 = _mm_load_ps(q1.w.data());

		auto x2 = _mm_load_ps(q2.x.data());
		auto y2 = _mm_load_ps(q2.y.data());
		auto z2 = _mm_load_ps(q2.z.data());
		auto w2 = _mm_load_ps(q2.w.data());

		x1 = _mm_mul_ps(x1, x2);
		y1 = _mm_mul_ps(y1, y2);
		z1 = _mm_mul_ps(z1, z2);
		w1 = _mm_mul_ps(w1, w2);

		x1 = _mm_add_ps(x1, y1);
		z1 = _mm_add_ps(z1, w1);
		x1 = _mm_add_ps(x1, z1);
		_mm_store_ps(result.data(), x1);
		return result;
	}
```
<!--- Show in code the implementation --->

Let me explain what the Intel Intrinsics function do:
### ps
ps means packed single_precision floating-points. It basically means 4 * 32 bit floating point numbers stored as a 128-bit value.

### _mm_load_ps()
The **_mm_load_ps** function is loading from memory the values contained in the array. The values must be 16 bytes which correspond to an *array* of 4 *float* numbers, and they must be aligned in memory, that’s why we’re using arrays that align the *floats* in memory.

### _mm_mul_ps()
The  **_mm_mul_ps** function is multiplying 4 *floats* with 4 other *floats*.

### _mm_add_ps()
The **_mm_add_ps** function is adding two values together.

### _mm_store_ps()
The **_mm_store_ps** function is storing from the value *x1* to the memory.

Here, instead of calculating the functions 4 times with 4 different Quaternions, I’ll be calculating them only once. This will allow to spare a lot of performance as I will show you.

## Performances
I created a test that calculated the **Dot product** of *n* quaternions using the MSVC compiler with a intel core i7 CPU on a windows 10 PC. Here’s the result:

<!--- Show a graphic of the code --->

As you can see, the **BM_Dot_Intrinsics** is between 3 and 4 times faster than the **BM_Dot** which is a huge performance optimisation.

<!--- Conclusion --->
