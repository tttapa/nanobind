# Object ownership and shared/unique pointers

When a C++ type is instantiated within Python via _nanobind_, the resulting
instance is stored _within_ the created Python object (henceforth `PyObject`).
Alternatively, when an already existing C++ instance is transferred to Python
via a function return value and `rv_policy::reference`,
`rv_policy::reference_internal`, or `rv_policy::take_ownership`, _nanobind_
creates a smaller `PyObject` that only stores a pointer to the instance data.

This is _very different_ from _pybind11_, where the instance `PyObject`
contained a _holder type_ (typically `std::unique_ptr<T>`) storing a pointer to
the instance data. Dealing with holders caused inefficiencies and introduced
complexity; they were therefore removed in _nanobind_. This has implications on
object ownership, shared ownership, and interactions with C++ shared/unique
pointers.

- **Shared pointers**: It is possible to bind functions that receive and return
  `std::shared_ptr<T>` by including the optional type caster
  [`nanobind/stl/shared_ptr.h`](https://github.com/wjakob/nanobind/blob/master/include/nanobind/stl/shared_ptr.h)
  in your code.

  When calling a C++ function with a `std::shared_ptr<T>` argument from Python,
  ownership must be shared between Python and C++. _nanobind_ does this by
  increasing the reference count of the `PyObject` and then creating a
  `std::shared_ptr<T>` with a new control block containing a custom deleter
  that reduces the Python reference count upon destruction of the shared
  pointer.

  When a C++ function returns a `std::shared_ptr<T>`, _nanobind_ checks if the
  instance already has a `PyObject` counterpart (nothing needs to be done in
  this case). Otherwise, it creates a compact `PyObject` wrapping a pointer to
  the instance data. It indicates shared ownership by creating a temporary
  `std::shared_ptr<T>` on the heap that will be destructed when the `PyObject`
  is garbage collected.

  Shared pointers therefore remain usable despite the lack of _holders_. The
  approach in _nanobind_ was chosen following on discussions with [Ralf
  Grosse-Kunstleve](https://github.com/rwgk); it is unusual in that multiple
  `shared_ptr` control blocks are potentially allocated for the same object,
  which means that `std::shared_ptr<T>::use_count()` generally won't show the
  true global reference count.

  One major limitation arises when `T` inherits from
  `std::enable_shared_from_this<T>`: in this case, it is **illegal** to create
  `std::shared_ptr<T>` from `T* value` if `value` is _owned by nanobind_. The
  shared pointer construtor would observe that no other shared pointers
  currently own `value` and incorrectly assume ownership of the instance. When
  the shared pointer expires, it will call `delete value`, causing undefined
  behavior that will likely crash your program (_nanobind_ allocates instances
  via _pymalloc_ as opposed to _new expressions_, so using `delete` in this
  context makes no sense). Calling
  ``std::enable_shared_from_this<T>::shared_from_this()`` within an instance
  owned by _nanobind_ will raise `std::bad_weak_ptr`.

  It is likely best to _refrain_ from using `std::enable_shared_from_this<T>`
  due to these limitations.

- **Unique pointers**: It is possible to bind functions that receive and return
  `std::unique_ptr<T, Deleter>` by including the optional type caster
  [`nanobind/stl/unique_ptr.h`](https://github.com/wjakob/nanobind/blob/master/include/nanobind/stl/unique_ptr.h)
  in your code.

  Whereas `std::shared_ptr<T>` could abstract over details concerning storage
  and the deletion mechanism, this is not possible in simpler
  `std::unique_ptr`, which means that some of those details leak into the type
  signature.

  When calling a C++ function with a `std::unique_ptr<T, Deleter>` argument
  from Python, there is an ownership transfer from Python to C++ that must be
  handled.

  * When `Deleter` is `std::default_delete<T>` (i.e., the default), this
    ownership transfer is only possible when the instance was originally
    created by a _new expression_ within C++ and _nanobind_ has taken over
    ownership (i.e., it was created by a function returning a raw pointer `T *`
    with `rv_policy::take_ownership` or a `std::unique_ptr<T>`). Otherwise,
    calling `delete` on the object will cause undefined behavior, and
    _nanobind_ therefore refuses the conversion with a warning.

  * To enable ownership transfer under all conditions, _nanobind_ provides a
    custom `Deleter` named `nb::deleter<T>`. It keeps the underlying `PyObject`
    alive during the lifetime of the unique pointer but marks it as invalid on
    the Python side, which means that operations involving the object will
    raise exceptions. Following this route requires changing the interface of
    all functions to be exposed in Python so that they take `std::unique_ptr<T,
    nb::deleter<T>>` as input (this custom deleter transparently handles _both_
    instances owned by C++ and instance owned by _nanobind_).

  Returning a `std::unique_ptr<T, Deleter>` from a function always works---but
  again, the details depend on the `Deleter`.

  * If `Deleter` is the default `std::default_delete<T>`, _nanobind_ ignores
    the return value policy (if specified) and takes ownership of the object.
    It will eventually invoke `delete` when the associated Python object is
    garbage-collected.

  * If `Deleter` is `nb::deleter<T>`, then _nanobind_ checks if the deleter
    indicates ownership by C++ or Python. In the former case, everything
    proceeds as with `std::default_deleter<T>`. Otherwise, a `PyObject` has
    already been created _nanobind_. This instance is currently marked as
    invalid and becomes valid again due to reverse ownership transfer.

  Deleters other than `std::default_delete<T>` or `nb::deleter<T>` are _not
  supported_.