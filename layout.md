---
sidebar_position: 2
---

# Fields (advanced)
Morden processor cores are orders of magnitude faster than their equipped memory systems. In quite a few scenarios, programs are bound by the speed of memory rather than cores. In order to shrink this speed gap, the multi-level cache system and high-bandwidth multi-channel memories are built into computer architectures.
Two questions frequently arises in making performance optimization efforts: 1) how to organize a faster data layout and 2) how to manage memory occupancy. In this article, we discuss the two questions in the scope of Taichi field. A better understanding of the mechanism under the hood is essential to write blazing fast Taichi programs.

If you feel confused about the basic concepts of Taichi, please refer to the basic document [Fields (basic)](../basic/field).


<!-- ## Data layouts -->
## How to organize an efficient data layout

<!-- Regarding this fact, Taichi's design puts great emphasis on the capability to organize data structures into efficient memory layouts. This is especially important when dealing with large fields. The performance of Taichi programs can benefit from a wisely-chosen field layout, and suffer from loosely organized buffers.  -->
<!-- This section is organized as follows: we first introduce the pricinples, then play basic Taichi , and at last ... -->
<!-- ### Basic Priniciples  -->
In this section, we introduce data layout organization in Taichi fields. The central principle of efficient data layout is locality. Generally speaking, a program with desirable locality has at least one of the following features: 
* Dense data structure 
* Loop over data in small-range (range within 32KB is especially good for most processors) 
* Sequential load/store. 

:::note

Brief explanation:

Be aware that data are always fetched from memory in blocks (pages). The hardware has no knowledge about the usefulness of a specific data element in the block. The processor just blindly fetch the entire block which _contains_ the requested memory address. Therefore, the memory bandwidth is wasted when the data in the block are not well utilized. 

If sparsity is inevitable, refer to [Sparse computation](./sparse.md).

:::

<!-- Apart from shape and data type, you can also specify the data layout of a
field in a recursive manner. This may allow you to achieve better performance.

Normally, you don't have to worry about the performance nuances between
different layouts, and you can just use the default one (simply by specifying
`shape` when creating fields) as a start.

However, when a field gets large, a proper data layout may be critical to
performance, especially for memory-bound applications. A carefully designed
data layout has much better spatial locality, which will significantly
improve cache/TLB-hit rates and cache line utilization.

Taichi decouples computation from data structures, and the Taichi compiler
automatically optimizes data accesses on a specific data layout. This allows
you to quickly experiment with different data layouts and figure out the most
efficient one on a specific task and computer architecture. -->


### Layout 101: from `shape` to `ti.root.X`

In basic usages, shape is regarded as an instant argument of field. However, advanced  organizations require more flexible descriptors, namely the `ti.root.X` statements.
<!-- haidong: what's else optional in ti.root? -->
Let's have some warmup practices to get formaliar to this set of statements. We strongly recommend that you reproduce the following code snippets in a Taichi environment.

* Declare a 0-D field:

```python {1-2}
x = ti.field(ti.f32)
ti.root.place(x)
# is equivalent to:
x = ti.field(ti.f32, shape=())
```

* Declare a 1D field of length `3`:

```python {1-2}
x = ti.field(ti.f32)
ti.root.dense(ti.i, 3).place(x)
# is equivalent to:
x = ti.field(ti.f32, shape=3)
```

* Declare a 2D field of shape `(3, 4)`:

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

In a nutshell, the `ti.root.X` statements gradually binds a shape to the corresponding axis.
Nested statements can compose a field with higher dimensions.

### Row-major versus column-major

As you might have learnt from a computer architecture course, memory address space is linear in modern computers. Without loss of generality, we ignore the differences in data types and always assume each data element has size 1. Denote the starting memory address of a field as `base`, the indexing formula for 1D Taichi fields is `base + i` for the `i`-th element.

For multi-dimensional fields, however, we have different ways to flatten the high-dimension index into the linear memory adress space. Take a 2D field of shape `(M, N)` as an instance, we can either store `M` rows with `N`-sized 1D buffers, say the row-major way, or store `N` columns, say the column-major way. The index flatten formula for the `(i, j)`-th element is `base + i * N + j` for row-major and `base + j * M + i` for column-major, respectively. 

We can easily derive that the elements in the same row are closer in memory for row-major fields, and the columns are more compact for column-major fields. The decision to use which layout is made by how the elements are accessed, namely, the access patterns. A commonly seen bad pattern is to frequently access elements of the same row in a column-major field, or the other way round.

The default Taichi field layout is row-major. With the `ti.root.dense` statement as we have practiced in the last section, the fields are defined as follows:
```python
ti.root.dense(ti.i, M).dense(ti.j, N).place(x)   # row-major
ti.root.dense(ti.j, N).dense(ti.i, M).place(y)   # column-major
```
The axis denotion in the rightmost `dense` statement indicates the axis which is continous in memory. For the `x` field, elements in the same row (same `i` and different `j`) are closer, hence it's row-major; For the `y` field, elements in the same column (same `j` and different `i`) are closer, hence it's column-major. With a case of (2, 3), we visualize the memory layouts of `x` and `y` in the memory as follows:
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
### AOS versus SOA

AOS means array of structures and SOA means structure of arrays. Consider an RGB image with 4 pixels and 3 color channels, the AOS layout stores `RGBRGBRGBRGB` and the SOA layout organizes the image as `RRRRGGGGBBBB`.

The selection of layouts largely depends on the access pattern to the field. Let's discuss the scenario of calculating grey scale of each pixel in a very large RGB image: it depends on all color channels but doesn't involve other pixels. The two layouts has the following arrangement in memory:
```
AOS: RGBRGBRGBRGBRGBRGB...
SOA: RRRRR...RGGGGGGG...GBBBBBBB...B
```
Apparently AOS layout has a straightened access fashion: all color channels are adjacent so they are obtained instantly. For the SOA layout, the color channels are pretty far away in the memory space. Therefore, the SOA layout is likely to be less efficient in this case.

 <!-- and good calulating the illumination of each pixel, AOS layout has a straight and plain access -->
<!-- Like our the row-major versus column-major  -->

We describe how to construct AOS and SOA fields with our `ti.root.X` statements. The SOA fields are trivial:
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

For AOS fields, we leverage the property that Taichi fields of the same shape can be compounded to a new field. With two elementary 1D fields `x` and `y` as an example, the AOS array can be constructed with 

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
That is to say, `place` interleaves the elements of Taichi fields `x` and `y`. The access methods to `x` and `y`, however, remain the same for AOS and SOA. We can change the data layout without revising the application logics. 

Let's consider a more specific case:

<!-- haiong: I hope this part is 1) revised to a runnable and complete example 2) provides performane constrast-->
```python
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

Take a simple 1D wave equation solver as an example:

```python
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

Here, `pos` and `vel` are placed separately, so the distance in address
space between `pos[i]` and `vel[i]` is `200000`. This results in poor spatial
locality and poor performance. A better way is to place them together:

```python
ti.root.dense(ti.i, N).place(pos, vel)
```

Then `vel[i]` is placed right next to `pos[i]`, which can increase spatial
locality and therefore improve performance.

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


### Flat versus hierarchical
<!-- haidong: I hope to remove this subsection. This content just repeats the AOS topic -->
Sometimes we want to access memory in a complex but fixed pattern, like traversing an image in 8x8 blocks. The apparent best practice is flatten each 8x8 block and concatenate them together. From a Taichi user's view, however, the field is no longer a flat buffer. It now has a hierarchy with two levels: the image level and the block level. The Taichi field can flexibly build the hierarchical fields with the `ti.root.X` statements:

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
where `M` and `N` are multiples of 8.

<!-- From the above discussions,

```python
val = ti.field(ti.f32, shape=(32, 64, 128))
```

is equivalent to the following in C/C++:

```c
float val[32][64][128];
``` -->

<!-- However, at times this data layout may be suboptimal for certain types of
computation tasks. For example, in trilinear texture interpolation,
`val[i, j, k]` and `val[i + 1, j, k]` are often accessed together. With the
above layout, they are very far away (32 KB) from each other, and not even
within the same 4 KB pages. This creates a huge cache/TLB pressure and leads
to poor performance. -->

<!-- A better layout might be

```python
val = ti.field(ti.f32)
ti.root.dense(ti.ijk, (8, 16, 32)).dense(ti.ijk, (4, 4, 4)).place(val)
```

This organizes `val` in `4x4x4` blocks, so that with high probability
`val[i, j, k]` and its neighbours are close to each other (i.e., in the
same cache line or memory page). -->

### Struct-fors

Struct-fors on nested dense data structures will automatically follow their
layout in memory. For example, if 2D scalar field `A` is defined in row-major,

```python
for i, j in A:
    A[i, j] += 1
```

will iterate over elements of `A` following the row-major order. Similarly, if
`A` is defined in column-major, then the iteration follows the column-major
order.

If `A` is hierarchical, it will be iterated level by level. This maximizes the
memory bandwidth utilization in most cases.

As you may notice, only dense data layouts are covered in this section. For sparse
data layouts, see

### Example code snippets

To end this section, we provide more example code snippets of advanced field data layout that might be useful in your Taichi program.

Row-major _(256, 256)_ 2D field:

```python
A = ti.field(ti.f32)
ti.root.dense(ti.ij, (256, 256)).place(A)
```

Column-major _(256, 256)_ 2D field:
```python
A = ti.field(ti.f32)
ti.root.dense(ti.j, 256).dense(ti.i, 256).place(A)
```

Hierarchical _(1024, 1024)_ 2D field with _8x8_ blocks:

```python
density = ti.field(ti.f32)
ti.root.dense(ti.ij, (128, 128)).dense(ti.ij, (8, 8)).place(density)
```

3D particle positions and velocities, AOS:

```python
pos = ti.Vector.field(3, dtype=ti.f32)
vel = ti.Vector.field(3, dtype=ti.f32)
ti.root.dense(ti.i, 1024).place(pos, vel)
# equivalent to
ti.root.dense(ti.i, 1024).place(pos.get_scalar_field(0),
                                pos.get_scalar_field(1),
                                pos.get_scalar_field(2),
                                vel.get_scalar_field(0),
                                vel.get_scalar_field(1),
                                vel.get_scalar_field(2))
```

3D particle positions and velocities, SOA:

```python
pos = ti.Vector.field(3, dtype=ti.f32)
vel = ti.Vector.field(3, dtype=ti.f32)
for i in range(3):
    ti.root.dense(ti.i, 1024).place(pos.get_scalar_field(i))
for i in range(3):
    ti.root.dense(ti.i, 1024).place(vel.get_scalar_field(i))
```

<!-- ## Memory Allocations -->
## How to manage memory occupancy
### Taichi field's default memory allocation strategies

### Dynamic field allocation and destruction

You can use the `FieldsBuilder` class for dynamic field allocation and destruction.
`FieldsBuilder` has the same data structure declaration APIs as `ti.root`,
including `dense()`. After declaration, you need to call the `finalize()`
method to compile it to an `SNodeTree` object.

A simple example is:

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

fb2 = ti.FieldsBuilder()
y = ti.field(dtype=ti.f32)
fb2.dense(ti.i, 5).place(y)
fb2_snode_tree = fb2.finalize()  # Finalizes the FieldsBuilder and returns a SNodeTree
func(y)
```

In fact, `ti.root` is implemented by `FieldsBuilder` implicitly, so you can
allocate the fields directly under `ti.root`:
```python
import taichi as ti
ti.init()  # Implicitly: ti.root = ti.FieldsBuilder()

@ti.kernel
def func(v: ti.template()):
    for I in ti.grouped(v):
        v[I] += 1

x = ti.field(dtype=ti.f32)
ti.root.dense(ti.ij, (5, 5)).place(x)
func(x)  # Automatically calls ti.root.finalize()
# Implicitly: ti.root = ti.FieldsBuilder()

y = ti.field(dtype=ti.f32)
ti.root.dense(ti.i, 5).place(y)
func(y)  # Automatically calls ti.root.finalize()
```

Furthermore, if you don't want to use the fields under a certain `SNodeTree`
anymore, you could call the `destroy()` method on the finalized `SNodeTree`
object, which will recycle its memory into the memory pool:

```py
import taichi as ti
ti.init()

@ti.kernel
def func(v: ti.template()):
    for I in ti.grouped(v):
        v[I] += 1

fb = ti.FieldsBuilder()
x = ti.field(dtype=ti.f32)
fb.dense(ti.ij, (5, 5)).place(x)
fb_snode_tree = fb.finalize()  # Finalizes the FieldsBuilder and returns a SNodeTree
func(x)

fb_snode_tree.destroy()  # x cannot be used anymore
```

### Packed mode
<!-- Haidong: I think this topic is not-so-advanced -->
The packed mode provides a compact memory allocation strategy. By default, Taichi implicitly fits a field in a larger buffer with power-of-two dimensions. For example, a `(18, 65)` field is materialized with an underlying `(32, 128)` buffer. The design enables fast indexing with bitwise arithmetics and better memory alignment. However, it definitely consumes more memory. The optional packed mode allocates buffer that tightly fits the requested field shape. It is especially useful when memory usage is a major concern.

In order to leverage the packed mode, spcifify the `packed` argument in `ti.init()`. The following code snippets demostrates the effects.
```python
ti.init()  # default: packed=False
a = ti.field(ti.i32, shape=(18, 65))  # padded to (32, 128)
```

```python
ti.init(packed=True)
a = ti.field(ti.i32, shape=(18, 65))  # no padding
```
