---
sidebar_position: 2
---

# Fields (advanced)

As is already widely recognized, memory access has great impact on program performance. Two frequently asked questions are: how to reduce memory occupancy and how to organize a faster data layout. In this article, we discuss the two questions in the scope of Taichi field. A better understanding of mechanism under the hood is essential to write blazing fast Taichi programs.

If you feel confused about the basic concepts of Taichi, please refer to the basic document [Fields (basic)](../basic/field).


## Data layouts

Morden processor cores are orders of magnitude faster than their equipped memory systems. In quite a few scenarios, programs are bound by the speed of memory rather than cores.
Regarding this fact, Taichi's design puts great emphasis on the capability to organize data structures into efficient memory layouts. This is especially important when dealing with large fields. The performance of Taichi programs can benefit from a wisely-chosen field layout, and suffer from loosely organized buffers. 

<!-- This section is organized as follows: we first introduce the pricinples, then play basic Taichi , and at last ... -->

### Basic Priniciples 
The central principle of buffer organization is locality. 

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
Let's have some warmup practices to get formaliar to this set of statements. We strongly recommend that you write following code snippets in a Taichi environment.

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
Nested statements can also compose a field with more dimensions.

### Row-major versus column-major

As you might have learnt from a computer architecture course, memory address space is linear in modern computers. Without loss of generality, we ignore the differences in data types and always assume each data element has size 1. Denote the starting memory address of a field as `base`, the indexing formula for 1D Taichi fields is `base + i` for the `i`-th element.

For multi-dimensional fields, however, we have different ways to compound the indices from multiple dimensions into the linear memory adress space. Take a 2D field of shape `(M, N)` as an instance, we can either store `M` rows with 1D buffers of length `N`, say the row-major way, or store `N` columns in 1D buffers of length `M`, say the column-major way. The indexing formulas for the 2D `(i, j)`-th element are `base + i * N + j` for row-major and `base + j * M + i` for column-major, respectively. 

We can easily derive that the elements in the same row are closer in memory for row-major fields, and the columns are more compact for column-major fields. The decision to use which layout is mainly made by how the elements are accessed, namely, the access patterns. A commonly seen bad pattern is to frequently access elements of the same row in a column-major field, or the other way round.

The default Taichi field layout is row-major. With the `ti.root.dense` statement as we have practiced in the last section, the fields are defined as follows:
```python
ti.root.dense(ti.i, M).dense(ti.j, N).place(x)   # row-major
ti.root.dense(ti.j, N).dense(ti.i, M).place(y)   # column-major
```
The axis denotion in the rightmost `dense` statement indicates the continous memory. For the `x` field, elements in the same row (same `i` and different `j`) are closer in memory, hence it's row-major; For the `y` field, elements in the same column (same `j` and different `i`) are closer in memory, hence it's column-major. With a case of (2, 3), we visualize the layouts of `x` and `y` in the memory as follows:
```
# address:  low ........................................... high
#       x:  x[0, 0]  x[0, 1]  x[1, 0]  x[1, 1]  x[2, 0]  x[2, 1]
#       y:  y[0, 0]  y[1, 0]  y[2, 0]  y[0, 1]  y[1, 1]  y[2, 1]
```

We specially note that the acessor is unified for Taichi fields: the `(i, j)`-th element in the field is accessed with the identical 2D index `x[i, j]` and `y[i, j]`. Taichi handles the layout variants internally and applies proper indexing equations. With this regard, the user can specify desired layout at definition, and use the fields with no awareness of underlying memory organizations. To change the layout, it's sufficient to swap the order of `dense` statements, and revise nothing in the later code sections.

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

AOS means array of structures and SOA means structure of arrays. Consider an RGB image with 4 pixels and 3 color channels, the AOS layout stores `RGBRGBRGBRGB` in the memory and the SOA layout the image as `RRRRGGGGBBBB`.

Fields of same shape can be placed together.

For example, the following places two 1D fields of shape `3` together, which
is called Array of Structures (AoS):

```python
ti.root.dense(ti.i, 3).place(x, y)
```

Their memory layout is:

```
#  address: low ......................... high
#           x[0]  y[0]  x[1]  y[1]  x[2]  y[2]
```

By contrast, the following places these two fields separately, which is called
Structure of Arrays (SoA):

```python
ti.root.dense(ti.i, 3).place(x)
ti.root.dense(ti.i, 3).place(y)
```

Now their memory layout is:

```
#  address: low ......................... high
#           x[0]  x[1]  x[2]  y[0]  y[1]  y[2]
```

**To improve spatial locality of memory accesses, it may be helpful to
place data elements that are often accessed together within
relatively close addresses.** Take a simple 1D wave equation solver as an example:

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

### Flat layouts versus hierarchical layouts

From the above discussions,

```python
val = ti.field(ti.f32, shape=(32, 64, 128))
```

is equivalent to the following in C/C++:

```c
float val[32][64][128];
```

However, at times this data layout may be suboptimal for certain types of
computation tasks. For example, in trilinear texture interpolation,
`val[i, j, k]` and `val[i + 1, j, k]` are often accessed together. With the
above layout, they are very far away (32 KB) from each other, and not even
within the same 4 KB pages. This creates a huge cache/TLB pressure and leads
to poor performance.

A better layout might be

```python
val = ti.field(ti.f32)
ti.root.dense(ti.ijk, (8, 16, 32)).dense(ti.ijk, (4, 4, 4)).place(val)
```

This organizes `val` in `4x4x4` blocks, so that with high probability
`val[i, j, k]` and its neighbours are close to each other (i.e., in the
same cache line or memory page).

### Struct-fors on advanced dense data layouts

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
data layouts, see [Sparse computation](./sparse.md).

### More examples of advanced dense data layouts

2D field, row-major:

```python
A = ti.field(ti.f32)
ti.root.dense(ti.ij, (256, 256)).place(A)
```

2D field, column-major:
```python
A = ti.field(ti.f32)
ti.root.dense(ti.j, 256).dense(ti.i, 256).place(A)
```

_8x8_-blocked 2D field of size _1024x1024_:

```python
density = ti.field(ti.f32)
ti.root.dense(ti.ij, (128, 128)).dense(ti.ij, (8, 8)).place(density)
```

3D particle positions and velocities, AoS:

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

3D particle positions and velocities, SoA:

```python
pos = ti.Vector.field(3, dtype=ti.f32)
vel = ti.Vector.field(3, dtype=ti.f32)
for i in range(3):
    ti.root.dense(ti.i, 1024).place(pos.get_scalar_field(i))
for i in range(3):
    ti.root.dense(ti.i, 1024).place(vel.get_scalar_field(i))
```

## Memory Allocations
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
