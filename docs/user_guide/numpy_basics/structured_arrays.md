# 结构化数组

## Introduction

结构化数组其实就是ndarrays，其数据类型是由组成一系列命名字段的简单数据类型组成的。 例如：

```python
>>> x = np.array([('Rex', 9, 81.0), ('Fido', 3, 27.0)],
...              dtype=[('name', 'U10'), ('age', 'i4'), ('weight', 'f4')])
>>> x
array([('Rex', 9, 81.0), ('Fido', 3, 27.0)],
      dtype=[('name', 'S10'), ('age', '<i4'), ('weight', '<f4')])
```

这里``x``是长度为2的一维数组，其数据类型是具有三个字段的结构：1、名为'name'的长度为10或更小的字符串。2、名为'age'的32位整数。3、名为'weight'的32位浮点数。

如果你的索引``x``是1，你会看到这样的结构：

```python
>>> x[1]
('Fido', 3, 27.0)
```

你可以通过使用字段名称进行索引来访问和修改结构化数组的各个字段的值：

```python
>>> x['age']
array([9, 3], dtype=int32)
>>> x['age'] = 5
>>> x
array([('Rex', 5, 81.0), ('Fido', 5, 27.0)],
      dtype=[('name', 'S10'), ('age', '<i4'), ('weight', '<f4')])
```

结构化数组设计用于结构化数据的底层操作，例如解释编译二进制数据块。结构化数据类型旨在模仿C语言中的 “structs”，使它们对于与C代码接口也很有用。 为了达到这些目的，numpy支持诸如子阵列和嵌套数据类型之类的特殊功能，并允许手动控制结构的内存布局。

如果你想进行表格数据的简单操作，那么其他 pydata 项目（例如pandas，xarray或DataArray）将为你提供更适合的更高级别的接口。 这些包也可以为表格数据分析提供更好的性能，因为NumPy中的结构化数组的类C结构内存布局会导致缓存行为不佳。

## 结构化数据类型

要使用结构化数组，首先需要定义结构化数据类型。

结构化数据类型可以被认为是一定长度的字节序列（结构的itemsize），它被解释为一个字段集合。 每个字段在结构中都有一个名称，一个数据类型和一个字节偏移量。 字段的数据类型可以是任何numpy数据类型，包括其他结构化数据类型，它也可以是一个子数组，其行为类似于指定形状的ndarray。 字段的偏移是任意的，并且字段甚至可以重叠。 这些偏移通常由numpy自动确定，但也可以指定。

### 结构化数据类型创建

结构化数据类型可以使用函数``numpy.dtype``创建。 有4种可选形式的规范，其灵活性和简洁性各不相同。 这些在[数据类型对象](https://docs.scipy.org/doc/numpy/reference/arrays.dtypes.html#arrays-dtypes-constructing)参考页面中都有进一步的记录，总之它们是：

1. **元组列表，每个字段一个元组**

    每个元组都有这些属性``（fieldname，datatype，shape）``，其中shape是可选的。 ``fieldname``是一个字符串（或元组，如果使用标题，请参阅下面的字段标题），``datatype``可以是任何可转换为数据类型的对象，``shape``是指定子阵列形状的整数元组。

    ```python
    >>> np.dtype([('x', 'f4'), ('y', np.float32), ('z', 'f4', (2,2))])
    dtype=[('x', '<f4'), ('y', '<f4'), ('z', '<f4', (2, 2))])
    ```

    如果``fieldname``是空字符串``''``，那么该字段将被赋予一个默认名称形式``f#``，其中``#``是该字段的整数索引，从左边以0开始计数：

    ```python
    >>> np.dtype([('x', 'f4'),('', 'i4'),('z', 'i8')])
    dtype([('x', '<f4'), ('f1', '<i4'), ('z', '<i8')])
    ```
    
    结构中字段的字节偏移量和总体结构中元素的大小是自动确定的。

1. **一串用逗号分隔的dtype规范**

    在这种简写表示法中，任何 [string dtype specifications](https://docs.scipy.org/doc/numpy/reference/arrays.dtypes.html#arrays-dtypes-constructing) 都可以在字符串中使用逗号分隔，字段的元素大小（itemsize）和字节偏移量是自动确定的，并且字段名称被赋予默认名称如":“f0”、“f1” 等。 

    ```python
    >>> np.dtype('i8,f4,S3')
    dtype([('f0', '<i8'), ('f1', '<f4'), ('f2', 'S3')])
    >>> np.dtype('3int8, float32, (2,3)float64')
    dtype([('f0', 'i1', 3), ('f1', '<f4'), ('f2', '<f8', (2, 3))])
    ```

3. **字段参数数组的字典**

    这是最灵活的规范形式，因为它允许控制字段的字节偏移量和结构中的元素大小（itemsize）。

    The dictionary has two required keys, ‘names’ and ‘formats’, and four optional keys, ‘offsets’, ‘itemsize’, ‘aligned’ and ‘titles’. The values for ‘names’ and ‘formats’ should respectively be a list of field names and a list of dtype specifications, of the same length. The optional ‘offsets’ value should be a list of integer byte-offsets, one for each field within the structure. If ‘offsets’ is not given the offsets are determined automatically. The optional ‘itemsize’ value should be an integer describing the total size in bytes of the dtype, which must be large enough to contain all the fields.

    ```python
    >>> np.dtype({'names': ['col1', 'col2'], 'formats': ['i4','f4']})
    dtype([('col1', '<i4'), ('col2', '<f4')])
    >>> np.dtype({'names': ['col1', 'col2'],
    ...           'formats': ['i4','f4'],
    ...           'offsets': [0, 4],
    ...           'itemsize': 12})
    dtype({'names':['col1','col2'], 'formats':['<i4','<f4'], 'offsets':[0,4], 'itemsize':12})
    ```

    Offsets may be chosen such that the fields overlap, though this will mean that assigning to one field may clobber any overlapping field’s data. As an exception, fields of ``numpy.object`` type cannot overlap with other fields, because of the risk of clobbering the internal object pointer and then dereferencing it.

    The optional ‘aligned’ value can be set to ``True`` to make the automatic offset computation use aligned offsets (see [Automatic Byte Offsets and Alignment](https://docs.scipy.org/doc/numpy/user/basics.rec.html#offsets-and-alignment)), as if the ‘align’ keyword argument of ``numpy.dtype`` had been set to True.

    The optional ‘titles’ value should be a list of titles of the same length as ‘names’, see [Field Titles](https://docs.scipy.org/doc/numpy/user/basics.rec.html#titles) below.

4. **字段名称的字典**

    The use of this form of specification is discouraged, but documented here because older numpy code may use it. The keys of the dictionary are the field names and the values are tuples specifying type and offset:

    ```python
    >>> np.dtype=({'col1': ('i1',0), 'col2': ('f4',1)})
    dtype([(('col1'), 'i1'), (('col2'), '>f4')])
    ```
    
    This form is discouraged because Python dictionaries do not preserve order in Python versions before Python 3.6, and the order of the fields in a structured dtype has meaning. [Field Titles](https://docs.scipy.org/doc/numpy/user/basics.rec.html#titles) may be specified by using a 3-tuple, see below.

### Manipulating and Displaying Structured Datatypes

The list of field names of a structured datatype can be found in the ``names`` attribute of the dtype object:

```python
>>> d = np.dtype([('x', 'i8'), ('y', 'f4')])
>>> d.names
('x', 'y')
```

The field names may be modified by assigning to the ``names`` attribute using a sequence of strings of the same length.

The dtype object also has a dictionary-like attribute, ``fields``, whose keys are the field names (and [Field Titles](https://docs.scipy.org/doc/numpy/user/basics.rec.html#titles), see below) and whose values are tuples containing the dtype and byte offset of each field.

```python
>>> d.fields
mappingproxy({'x': (dtype('int64'), 0), 'y': (dtype('float32'), 8)})
```

Both the ``names`` and ``fields`` attributes will equal ``None`` for unstructured arrays.

The string representation of a structured datatype is shown in the “list of tuples” form if possible, otherwise numpy falls back to using the more general dictionary form.

### Manipulating and Displaying Structured Datatypes

Numpy uses one of two methods to automatically determine the field byte offsets and the overall itemsize of a structured datatype, depending on whether ``align=True`` was specified as a keyword argument to ``numpy.dtype``.

By default (``align=False``), numpy will pack the fields together such that each field starts at the byte offset the previous field ended, and the fields are contiguous in memory.

```python
>>> def print_offsets(d):
...     print("offsets:", [d.fields[name][1] for name in d.names])
...     print("itemsize:", d.itemsize)
>>> print_offsets(np.dtype('u1,u1,i4,u1,i8,u2'))
offsets: [0, 1, 2, 6, 7, 15]
itemsize: 17
```

If ``align=True`` is set, numpy will pad the structure in the same way many C compilers would pad a C-struct. Aligned structures can give a performance improvement in some cases, at the cost of increased datatype size. Padding bytes are inserted between fields such that each field’s byte offset will be a multiple of that field’s alignment, which is usually equal to the field’s size in bytes for simple datatypes, see ``PyArray_Descr.alignment``. The structure will also have trailing padding added so that its itemsize is a multiple of the largest field’s alignment.

```python
>>> print_offsets(np.dtype('u1,u1,i4,u1,i8,u2', align=True))
offsets: [0, 1, 4, 8, 16, 24]
itemsize: 32
```

Note that although almost all modern C compilers pad in this way by default, padding in C structs is C-implementation-dependent so this memory layout is not guaranteed to exactly match that of a corresponding struct in a C program. Some work may be needed, either on the numpy side or the C side, to obtain exact correspondence.

If offsets were specified using the optional ``offsets`` key in the dictionary-based dtype specification, setting ``align=True`` will check that each field’s offset is a multiple of its size and that the itemsize is a multiple of the largest field size, and raise an exception if not.

If the offsets of the fields and itemsize of a structured array satisfy the alignment conditions, the array will have the ``ALIGNED`` flag set.

A convenience function ``numpy.lib.recfunctions.repack_fields`` converts an aligned dtype or array to a packed one and vice versa. It takes either a dtype or structured ndarray as an argument, and returns a copy with fields re-packed, with or without padding bytes.

### Field Titles

In addition to field names, fields may also have an associated title, an alternate name, which is sometimes used as an additional description or alias for the field. The title may be used to index an array, just like a field name.

To add titles when using the list-of-tuples form of dtype specification, the field name may be be specified as a tuple of two strings instead of a single string, which will be the field’s title and field name respectively. For example:

```python
>>> np.dtype([(('my title', 'name'), 'f4')])
```

When using the first form of dictionary-based specification, the ``titles`` may be supplied as an extra 'titles' key as described above. When using the second (discouraged) dictionary-based specification, the title can be supplied by providing a 3-element tuple ``(datatype, offset, title)`` instead of the usual 2-element tuple:

```python
>>> np.dtype({'name': ('i4', 0, 'my title')})
```

The ``dtype.fields`` dictionary will contain titles as keys, if any titles are used. This means effectively that a field with a title will be represented twice in the fields dictionary. The tuple values for these fields will also have a third element, the field title. Because of this, and because the ``names`` attribute preserves the field order while the ``fields`` attribute may not, it is recommended to iterate through the fields of a dtype using the ``names`` attribute of the dtype, which will not list titles, as in:

```python
>>> for name in d.names:
...     print(d.fields[name][:2])
```

### Union types

Structured datatypes are implemented in numpy to have base type ``numpy.void`` by default, but it is possible to interpret other numpy types as structured types using the ``(base_dtype, dtype)`` form of dtype specification described in Data Type Objects. Here, ``base_dtype`` is the desired underlying dtype, and fields and flags will be copied from ``dtype``. This dtype is similar to a ‘union’ in C.

## Indexing and Assignment to Structured arrays

### Assigning data to a Structured Array

There are a number of ways to assign values to a structured array: Using python tuples, using scalar values, or using other structured arrays.

#### Assignment from Python Native Types (Tuples)

The simplest way to assign values to a structured array is using python tuples. Each assigned value should be a tuple of length equal to the number of fields in the array, and not a list or array as these will trigger numpy’s broadcasting rules. The tuple’s elements are assigned to the successive fields of the array, from left to right:

```python
>>> x = np.array([(1,2,3),(4,5,6)], dtype='i8,f4,f8')
>>> x[1] = (7,8,9)
>>> x
array([(1, 2., 3.), (7, 8., 9.)],
     dtype=[('f0', '<i8'), ('f1', '<f4'), ('f2', '<f8')])
```

#### Assignment from Scalars

A scalar assigned to a structured element will be assigned to all fields. This happens when a scalar is assigned to a structured array, or when an unstructured array is assigned to a structured array:

```python
>>> x = np.zeros(2, dtype='i8,f4,?,S1')
>>> x[:] = 3
>>> x
array([(3, 3.0, True, b'3'), (3, 3.0, True, b'3')],
      dtype=[('f0', '<i8'), ('f1', '<f4'), ('f2', '?'), ('f3', 'S1')])
>>> x[:] = np.arange(2)
>>> x
array([(0, 0.0, False, b'0'), (1, 1.0, True, b'1')],
      dtype=[('f0', '<i8'), ('f1', '<f4'), ('f2', '?'), ('f3', 'S1')])
```

Structured arrays can also be assigned to unstructured arrays, but only if the structured datatype has just a single field:

```python
>>> twofield = np.zeros(2, dtype=[('A', 'i4'), ('B', 'i4')])
>>> onefield = np.zeros(2, dtype=[('A', 'i4')])
>>> nostruct = np.zeros(2, dtype='i4')
>>> nostruct[:] = twofield
ValueError: Can't cast from structure to non-structure, except if the structure only has a single field.
>>> nostruct[:] = onefield
>>> nostruct
array([0, 0], dtype=int32)
```

#### Assignment from other Structured Arrays

Assignment between two structured arrays occurs as if the source elements had been converted to tuples and then assigned to the destination elements. That is, the first field of the source array is assigned to the first field of the destination array, and the second field likewise, and so on, regardless of field names. Structured arrays with a different number of fields cannot be assigned to each other. Bytes of the destination structure which are not included in any of the fields are unaffected.

```python
>>> a = np.zeros(3, dtype=[('a', 'i8'), ('b', 'f4'), ('c', 'S3')])
>>> b = np.ones(3, dtype=[('x', 'f4'), ('y', 'S3'), ('z', 'O')])
>>> b[:] = a
>>> b
array([(0.0, b'0.0', b''), (0.0, b'0.0', b''), (0.0, b'0.0', b'')],
      dtype=[('x', '<f4'), ('y', 'S3'), ('z', 'O')])
```

#### Assignment involving subarrays

When assigning to fields which are subarrays, the assigned value will first be broadcast to the shape of the subarray.

### Indexing Structured Arrays

#### Accessing Individual Fields

Individual fields of a structured array may be accessed and modified by indexing the array with the field name.

```python
>>> x = np.array([(1,2),(3,4)], dtype=[('foo', 'i8'), ('bar', 'f4')])
>>> x['foo']
array([1, 3])
>>> x['foo'] = 10
>>> x
array([(10, 2.), (10, 4.)],
      dtype=[('foo', '<i8'), ('bar', '<f4')])
```

The resulting array is a view into the original array. It shares the same memory locations and writing to the view will modify the original array.

```python
>>> y = x['bar']
>>> y[:] = 10
>>> x
array([(10, 5.), (10, 5.)],
      dtype=[('foo', '<i8'), ('bar', '<f4')])
```

This view has the same dtype and itemsize as the indexed field, so it is typically a non-structured array, except in the case of nested structures.

```python
>>> y.dtype, y.shape, y.strides
(dtype('float32'), (2,), (12,))
```

#### Accessing Multiple Fields

One can index and assign to a structured array with a multi-field index, where the index is a list of field names.

<div class="warning-warp">
<b>Warning</b>

<p>The behavior of multi-field indexes will change from Numpy 1.14 to Numpy 1.15.</p>
</div>

In Numpy 1.15, the result of indexing with a multi-field index will be a view into the original array, as follows:

```python
>>> a = np.zeros(3, dtype=[('a', 'i4'), ('b', 'i4'), ('c', 'f4')])
>>> a[['a', 'c']]
array([(0, 0.), (0, 0.), (0, 0.)],
     dtype={'names':['a','c'], 'formats':['<i4','<f4'], 'offsets':[0,8], 'itemsize':12})
```

Assignment to the view modifies the original array. The view’s fields will be in the order they were indexed. Note that unlike for single-field indexing, the view’s dtype has the same itemsize as the original array, and has fields at the same offsets as in the original array, and unindexed fields are merely missing.

In Numpy 1.14, indexing an array with a multi-field index returns a copy of the result above for 1.15, but with fields packed together in memory as if passed through ``numpy.lib.recfunctions.repack_fields``. This is the behavior of Numpy 1.7 to 1.13.

<div class="warning-warp">
<b>Warning</b>
<p>The new behavior in Numpy 1.15 leads to extra “padding” bytes at the location of unindexed fields. You will need to update any code which depends on the data having a “packed” layout. For instance code such as:</p>
<pre class="prettyprint language-python">
<code class="hljs">>>> a[['a','c']].view('i8')  # will fail in Numpy 1.15
ValueError: When changing to a smaller dtype, its size must be a divisor of the size of original dtype</code>
</pre>
<p>will need to be changed. This code has raised a <code>FutureWarning</code> since Numpy 1.12.</p>
<p>The following is a recommended fix, which will behave identically in Numpy 1.14 and Numpy 1.15:</p>
<pre class="prettyprint language-python">
<code class="hljs">>>> from numpy.lib.recfunctions import repack_fields
>>> repack_fields(a[['a','c']]).view('i8')  # supported 1.14 and 1.15
array([0, 0, 0])</code>
</pre>
</div>

Assigning to an array with a multi-field index will behave the same in Numpy 1.14 and Numpy 1.15. In both versions the assignment will modify the original array:

```python
>>> a[['a', 'c']] = (2, 3)
>>> a
array([(2, 0, 3.0), (2, 0, 3.0), (2, 0, 3.0)],
      dtype=[('a', '<i8'), ('b', '<i4'), ('c', '<f8')])
```

This obeys the structured array assignment rules described above. For example, this means that one can swap the values of two fields using appropriate multi-field indexes:

```python
>>> a[['a', 'c']] = a[['c', 'a']]
```

#### Indexing with an Integer to get a Structured Scalar

Indexing a single element of a structured array (with an integer index) returns a structured scalar:

```python
>>> x = np.array([(1, 2., 3.)], dtype='i,f,f')
>>> scalar = x[0]
>>> scalar
(1, 2., 3.)
>>> type(scalar)
numpy.void
```

Unlike other numpy scalars, structured scalars are mutable and act like views into the original array, such that modifying the scalar will modify the original array. Structured scalars also support access and assignment by field name:

```python
>>> x = np.array([(1,2),(3,4)], dtype=[('foo', 'i8'), ('bar', 'f4')])
>>> s = x[0]
>>> s['bar'] = 100
>>> x
array([(1, 100.), (3, 4.)],
      dtype=[('foo', '<i8'), ('bar', '<f4')])
```

Similarly to tuples, structured scalars can also be indexed with an integer:

```python
>>> scalar = np.array([(1, 2., 3.)], dtype='i,f,f')[0]
>>> scalar[0]
1
>>> scalar[1] = 4
```

Thus, tuples might be thought of as the native Python equivalent to numpy’s structured types, much like native python integers are the equivalent to numpy’s integer types. Structured scalars may be converted to a tuple by calling ndarray.item:

```python
>>> scalar.item(), type(scalar.item())
((1, 2.0, 3.0), tuple)
```

### Viewing Structured Arrays Containing Objects

In order to prevent clobbering object pointers in fields of ``numpy.object`` type, numpy currently does not allow views of structured arrays containing objects.

### Structure Comparison

If the dtypes of two void structured arrays are equal, testing the equality of the arrays will result in a boolean array with the dimensions of the original arrays, with elements set to True where all fields of the corresponding structures are equal. Structured dtypes are equal if the field names, dtypes and titles are the same, ignoring endianness, and the fields are in the same order:

```python
>>> a = np.zeros(2, dtype=[('a', 'i4'), ('b', 'i4')])
>>> b = np.ones(2, dtype=[('a', 'i4'), ('b', 'i4')])
>>> a == b
array([False, False])
```

Currently, if the dtypes of two void structured arrays are not equivalent the comparison fails, returning the scalar value ``False``. This behavior is deprecated as of numpy 1.10 and will raise an error or perform elementwise comparison in the future.

The ``<`` and ``>`` operators always return ``False`` when comparing void structured arrays, and arithmetic and bitwise operations are not supported.

## Record Arrays

As an optional convenience numpy provides an ndarray subclass, ``numpy.recarray``, and associated helper functions in the ``numpy.rec`` submodule, that allows access to fields of structured arrays by attribute instead of only by index. Record arrays also use a special datatype, ``numpy.record``, that allows field access by attribute on the structured scalars obtained from the array.

The simplest way to create a record array is with ``numpy.rec.array``:

```python
>>> recordarr = np.rec.array([(1,2.,'Hello'),(2,3.,"World")],
...                    dtype=[('foo', 'i4'),('bar', 'f4'), ('baz', 'S10')])
>>> recordarr.bar
array([ 2.,  3.], dtype=float32)
>>> recordarr[1:2]
rec.array([(2, 3.0, 'World')],
      dtype=[('foo', '<i4'), ('bar', '<f4'), ('baz', 'S10')])
>>> recordarr[1:2].foo
array([2], dtype=int32)
>>> recordarr.foo[1:2]
array([2], dtype=int32)
>>> recordarr[1].baz
'World'
```

``numpy.rec.array`` can convert a wide variety of arguments into record arrays, including structured arrays:

```python
>>> arr = array([(1,2.,'Hello'),(2,3.,"World")],
...             dtype=[('foo', 'i4'), ('bar', 'f4'), ('baz', 'S10')])
>>> recordarr = np.rec.array(arr)
```

The ``numpy.rec`` module provides a number of other convenience functions for creating record arrays, see [record array creation routines](https://docs.scipy.org/doc/numpy/reference/routines.array-creation.html#routines-array-creation-rec).

A record array representation of a structured array can be obtained using the appropriate view:

```python
>>> arr = np.array([(1,2.,'Hello'),(2,3.,"World")],
...                dtype=[('foo', 'i4'),('bar', 'f4'), ('baz', 'a10')])
>>> recordarr = arr.view(dtype=dtype((np.record, arr.dtype)),
...                      type=np.recarray)
```

For convenience, viewing an ndarray as type ``np.recarray`` will automatically convert to ``np.record`` datatype, so the dtype can be left out of the view:

```python
>>> recordarr = arr.view(np.recarray)
>>> recordarr.dtype
dtype((numpy.record, [('foo', '<i4'), ('bar', '<f4'), ('baz', 'S10')]))
```

To get back to a plain ndarray both the dtype and type must be reset. The following view does so, taking into account the unusual case that the recordarr was not a structured type:

```python
>>> arr2 = recordarr.view(recordarr.dtype.fields or recordarr.dtype, np.ndarray)
```

Record array fields accessed by index or by attribute are returned as a record array if the field has a structured type but as a plain ndarray otherwise.

```python
>>> recordarr = np.rec.array([('Hello', (1,2)),("World", (3,4))],
...                 dtype=[('foo', 'S6'),('bar', [('A', int), ('B', int)])])
>>> type(recordarr.foo)
<type 'numpy.ndarray'>
>>> type(recordarr.bar)
<class 'numpy.core.records.recarray'>
```

Note that if a field has the same name as an ndarray attribute, the ndarray attribute takes precedence. Such fields will be inaccessible by attribute but will still be accessible by index.