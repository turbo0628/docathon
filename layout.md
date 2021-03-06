---
sidebar_position: 2
---

# Fields (advanced)
Morden processor cores are orders of magnitude faster than their equipped memory systems. In quite a few scenarios, programs are bound by the speed of memory rather than cores. In order to shrink this speed gap, the multi-level cache system and high-bandwidth multi-channel memories are built into computer architectures.

There arises two questions in making performance optimization efforts: 1) how to organize a faster data layout and 2) how to manage memory occupancy. In this article, we discuss in the scope of Taichi field. A better understanding of the mechanism under the hood is essential to write blazing fast Taichi programs.

If you feel confused about the basic concepts of Taichi, please refer to the basic document [Fields (basic)](../basic/field).


## How to organize an efficient data layout

In this section, we introduce how to organize data layouts in Taichi fields. The central principle of efficient data layout is locality. Generally speaking, a program with desirable locality has at least one of the following features: 
* Dense data structure 
* Loop over data in small-range (range within 32KB is especially good for most processors) 
* Sequential load/store

:::note

Brief explanations:

Be aware that data are always fetched from memory in blocks (pages). The hardware has no knowledge about the usefulness of one specific data element in the block. The processor blindly fetch the entire block which _contains_ the requested memory address. Therefore, the memory bandwidth is wasted when data in the block are not well utilized. 

If sparsity is inevitable, refer to [Sparse computation](./sparse.md).

:::

### Layout 101: from `shape` to `ti.root.X`

<!-- haidong: what's else optional in ti.root? -->
In basic usages, we use the `shape` descriptor to construct a field. Taichi provides flexible statements to describe more advanced data organizations, the `ti.root.X`.
Let's get some formiliarity with examples:

* Declare a 0-D field:

```python {1-2}
x = ti.field(ti.f32)
ti.root.place(x)
# is equivalent to:
x = ti.field(ti.f32, shape=())
```

* Declare a 1-D field of length `3`:

```python {1-2}
x = ti.field(ti.f32)
ti.root.dense(ti.i, 3).place(x)
# is equivalent to:
x = ti.field(ti.f32, shape=3)
```

* Declare a 2-D field of shape `(3, 4)`:

```python {1-2}
x = ti.field(ti.f32)
ti.root.dense(ti.ij, (3, 4)).place(x)
# is equivalent to:
x = ti.field(ti.f32, shape=(3, 4))
```
You can also nest two 1D `dense` statements to describe the same 2D array.
```python {1-2}
x = ti.field(ti.f32)
ti.root.dense(ti.i, 3).dense(ti.j, 4).place(x)
```

In a nutshell, the `ti.root.X` statements progressively binds a shape to the corresponding axis.
By nesting multiple statements, we can construct a field with higher dimensions.
<!-- haidong: how far can we go? how many default axis exist? -->

In order to traverse the nested statements, we can use `struct-for`:
```python
for i, j in A:
    A[i, j] += 1
```
The order to access `A`, namely the order to iterate `i` and `j`, affects the program performance subtly. In general programming language, we need to optimize the order manually. The Taichi compiler is capable to deduce underlying data layout and apply proper access order. Therefore, the access order to complex nested structures is implicitly optimized.

### Row-major versus column-major

Memory address space is linear as you might have learnt from a computer architecture course. Without loss of generality, we ignore the differences in data types and always assume each data element has size 1. Denote the starting memory address of a field as `base`, the indexing formula for 1D Taichi fields is `base + i` for the `i`-th element.

For multi-dimensional fields, however, we have different ways to flatten the high-dimension index into the linear memory adress space. Take a 2D field of shape `(M, N)` as an instance, we can either store `M` rows with `N`-length 1D buffers, say the row-major way, or store `N` columns, say the column-major way. The index flatten formula for the `(i, j)`-th element is `base + i * N + j` for row-major and `base + j * M + i` for column-major, respectively. 

We can easily derive that the elements in the same row are closer in memory for row-major fields. The selection of optimal layout is made by how the elements are accessed, namely, the access patterns. A commonly seen bad pattern is to frequently access elements of the same row in a column-major field, or the other way round.

The default Taichi field layout is row-major. With the `ti.root` statements, the fields can be defined as follows:
```python
ti.root.dense(ti.i, M).dense(ti.j, N).place(x)   # row-major
ti.root.dense(ti.j, N).dense(ti.i, M).place(y)   # column-major
```
The axis denotion in the rightmost `dense` statement indicates the continous axis. For the `x` field, elements in the same row (same `i` and different `j`) are closer, hence it's row-major; For the `y` field, elements in the same column (same `j` and different `i`) are closer, hence it's column-major. With an example of (2, 3), we visualize the memory layouts of `x` and `y` as follows:
```
# address:  low ........................................... high
#       x:  x[0, 0]  x[0, 1]  x[1, 0]  x[1, 1]  x[2, 0]  x[2, 1]
#       y:  y[0, 0]  y[1, 0]  y[2, 0]  y[0, 1]  y[1, 1]  y[2, 1]
```

We specially note that the acessor is unified for Taichi fields: the `(i, j)`-th element in the field is accessed with the identical 2D index `x[i, j]` and `y[i, j]`. Taichi handles the layout variants and applies proper indexing equations internally. With this regard, users can specify desired layout at definition, and use the fields without concerning about underlying memory organizations. To change the layout, it's sufficient to swap the order of `dense` statements, and leave the later code sections unchanged.

:::note

For readers who are familiar with C/C++, below is an example C code snippet that demonstrates data access in 2D arrays:
```c
int x[3][2];  // row-major
int y[2][3];  // column-major

for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 2; j++) {
        do_something(x[i][j]);
        do_something(y[j][i]);
    }
}
```

The accessors of `x` and `y` are in reverse order for row-major arrays and colomn-major arrays, respectively. Compared with Taichi fields, there are much more code to revise when you change the memory layout.

:::

<!-- ### Array of Structures (AoS) versus Structure of Arrays (SoA) -->
### AoS versus SoA

AoS means array of structures and SoA means structure of arrays. Consider an RGB image with 4 pixels and 3 color channels, the AoS layout stores `RGBRGBRGBRGB` and the SoA layout stores `RRRRGGGGBBBB`.

The selection of AoS or SoA layouts largely depends on the access pattern to the field. Let's discuss the scenario to process large RGB images. The two layouts has the following arrangement in memory:
```
# address: low ...................... high
# AoS:     RGBRGBRGBRGBRGBRGB.............
# SoA:     RRRRR...RGGGGGGG...GBBBBBBB...B
```
To calculate grey scale of each pixel, we need all color channels but doesn't involve other pixels. Apparently AoS layout has a better access fashion: all color channels are adjacent so they are obtained instantly while the color channels are pretty far away for the SoA layout. However, if we compute the mean value of each color channel, then SoA is the more efficient choice. 

We describe how to construct AoS and SoA fields with our `ti.root.X` statements. The SoA fields are trivial:
```python
x = ti.field(ti.f32)
y = ti.field(ti.f32)
ti.root.dense(ti.i, M).place(x)
ti.root.dense(ti.i, M).place(y)
```
where M is the length of `x` and `y`. 
The data elements in `x` and `y` are naturally continuous in memory:
```
#  address: low ................................. high
#           x[0]  x[1]  x[2] ... y[0]  y[1]  y[2] ...
```

For AoS fields, we construct the field with 

```python
x = ti.field(ti.f32)
y = ti.field(ti.f32)
ti.root.dense(ti.i, M).place(x, y)
```
The memroy layout then becomes
```
#  address: low .............................. high
#           x[0]  y[0]  x[1]  y[1]  x[2]  y[2] ...
```
Here, `place` interleaves the elements of Taichi fields `x` and `y`. 

The access methods to `x` and `y`, however, remain the same for AoS and SoA. We can change the data layout without revising the application logics. 

<!-- haiong: I hope this part is 1) revised to a runnable and complete example 2) provides performane constrast-->
To be more clear, let's see a specific example, 1D wave equation solver:
```python
N = 200000
pos = ti.field(ti.f32)
vel = ti.field(ti.f32)
# SoA placement
ti.root.dense(ti.i, N).place(pos)
ti.root.dense(ti.i, N).place(vel)

@ti.kernel
def step():
    pos[i] += vel[i] * dt
    vel[i] += -k * pos[i] * dt

...
```
The above code snippet defines SoA fields and sequentially access each element in the subsequent `step` kernel.
The kernel fetches an element from `pos` and `vel` for every iteration, respectively.
For SoA fields, the closest distance of the two elements in memory is `N`, which is unlikely to be efficient for large `N`.

We hereby switch the layout to AoS as follows: 

```python
N = 200000
pos = ti.field(ti.f32)
vel = ti.field(ti.f32)
# AoS placement
ti.root.dense(ti.i, N).place(pos, vel)

@ti.kernel
def step():
    pos[i] += vel[i] * dt
    vel[i] += -k * pos[i] * dt
```
Merely revising the place statement is sufficient to change the layout. With this regard, the instant elements `pos[i]` and `vel[i]` are now adjecent in memory, which appears to be more efficient.

<!-- ```python
# SoA version
N = 200000
pos = ti.field(ti.f32)
vel = ti.field(ti.f32)
ti.root.dense(ti.i, N).place(pos)
ti.root.dense(ti.i, N).place(vel)

@ti.kernel
def step():
    pos[i] += vel[i] * dt
    vel[i] += -k * pos[i] * dt
```


```python
# AoS version
N = 200000
pos = ti.field(ti.f32)
vel = ti.field(ti.f32)
ti.root.dense(ti.i, N).place(pos)
ti.root.dense(ti.i, N).place(vel)

@ti.kernel
def step():
    pos[i] += vel[i] * dt
    vel[i] += -k * pos[i] * dt
```

Here, `pos` and `vel` for SoA are placed separately, so the distance in address
space between `pos[i]` and `vel[i]` is `200000`. This results in poor spatial
locality and poor performance. A better way is to place them together:

```python
ti.root.dense(ti.i, N).place(pos, vel)
```

Then `vel[i]` is placed right next to `pos[i]`, which can increase spatial
locality and therefore improve performance. -->

<!-- For example, the following places two 1D fields of shape `3` together, which
is called Array of Structures (AoS):

```python
ti.root.dense(ti.i, 3).place(x, y)
```

Their memory layout is:

By contrast, the following places these two fields separately, which is called
Structure of Arrays (SoA): -->

<!-- ```python
ti.root.dense(ti.i, 3).place(x)
ti.root.dense(ti.i, 3).place(y)
```

Now their memory layout is:


**To improve spatial locality of memory accesses, it may be helpful to
place data elements that are often accessed together within
relatively close addresses.**  -->


### AoS extension: hierarchical fields
<!-- haidong: I hope to remove this subsection. This content just repeats the AoS topic -->
Sometimes we want to access memory in a complex but fixed pattern, like traversing an image in 8x8 blocks. The apparent best practice is to flatten each 8x8 block and concatenate them together. From a Taichi user's view, however, the field is no longer a flat buffer. It now has a hierarchy with two levels: the image level and the block level. Equivalently, the field is an array of implicit 8x8 block strucutrues.

We demonstrate the statements as follows:

```python
# Flat field
val = ti.field(ti.f32)
ti.root.dense(ti.ij, (M, N)).place(val)
```

```python
# Hierarchical field
val = ti.field(ti.f32)
ti.root.dense(ti.ij, (M // 8, N // 8)).dense(ti.ij, (8, 8)).place(val)
```
where `M` and `N` are multiples of 8. We hope the readers try on their own compute machines. The performance difference can be amazing.

## How to manage memory occupancy

### Manual field allocation and destruction

Generally Taichi manages memory allocation and destruction without disturbing the users. However, there are sometimes that users want delicate controll over memory allocations. 

In this scenario, Taichi provides the `FieldsBuilder` for manual field memory allocation and destruction. `FieldsBuilder` features identical declaration APIs as `ti.root`. The extra step is to invoke `finalize()` at the end of all delarations. The `finalize()` returns an `SNodeTree` object to handle subsequent destructions.

Let's see a simple example:

```python
import taichi as ti
ti.init()

@ti.kernel
def func(v: ti.template()):
    for I in ti.grouped(v):
        v[I] += 1

fb1 = ti.FieldsBuilder()
x = ti.field(dtype=ti.f32)
fb1.dense(ti.ij, (5, 5)).place(x)
fb1_snode_tree = fb1.finalize()  # Finalizes the FieldsBuilder and returns a SNodeTree
func(x)
...
fb1_snode_tree.destroy() # Destruction

fb2 = ti.FieldsBuilder()
y = ti.field(dtype=ti.f32)
fb2.dense(ti.i, 5).place(y)
fb2_snode_tree = fb2.finalize()  # Finalizes the FieldsBuilder and returns a SNodeTree
func(y)
...
fb2_snode_tree.destroy() # Destruction
```
Actually, the above demonstrated `ti.root` statements are implemented with `FieldsBuilder`, despite that `ti.root` has the capability to automatically manage memory allocations and recycling.

### Packed mode

By default, Taichi implicitly fits a field in a larger buffer with power-of-two dimensions. We take the power-of-two padding convention as it is widely adopted in computer graphics. The design enables fast indexing with bitwise arithmetics and better memory address alignment, while trades off memory occupations especially for large irregular fields.

For example, a `(18, 65)` field is materialized with a `(32, 128)` buffer, which is acceptable. As field size grows, the padding stragtegy can be exaggeratedly unbearable: `(129, 6553600)` will be padded to `(256, 6335600)`, which is a waste of the precious memory capacity. Therefore, Taichi provides the optional packed mode to allocate buffer that tightly fits the requested field shape. It is especially useful when memory usage is a major concern.

To leverage the packed mode, spcifify `packed` in `ti.init()` argument:
```python
ti.init()  # default: packed=False
a = ti.field(ti.i32, shape=(18, 65))  # padded to (32, 128)
```

```python
ti.init(packed=True)
a = ti.field(ti.i32, shape=(18, 65))  # no padding
```

You might obeserve slight performance regression, which is not a big concern in most cases.