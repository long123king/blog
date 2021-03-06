# Python Call Implementation

## PyObject_Call的实现
在builtin_map函数中，我们看到，最后是调用了

        value = PyEval_CallObject(func, alist);
来依次处理每个元素，下面是调用层次：

        --> PyEval_CallObject
           |--> PyEval_CallObjectWithKeywords
              |--> PyObject_Call


        PyObject *
        PyObject_Call(PyObject *func, PyObject *arg, PyObject *kw)
        {
            ternaryfunc call;

            if ((call = func->ob_type->tp_call) != NULL) {
                PyObject *result;
                if (Py_EnterRecursiveCall(" while calling a Python object"))
                    return NULL;
                result = (*call)(func, arg, kw);
                Py_LeaveRecursiveCall();
                if (result == NULL && !PyErr_Occurred())
                    PyErr_SetString(
                        PyExc_SystemError,
                        "NULL result without error in PyObject_Call");
                return result;
            }
            PyErr_Format(PyExc_TypeError, "'%.200s' object is not callable",
                         func->ob_type->tp_name);
            return NULL;
        } 

首先，我们看到这一段代码

        (call = func->ob_type->tp_call) != NULL

可见，type类型对象中的tp_call成员，是一个Callable Object的必须填写的字段，当这个Callable Object被调用时，
就会直接调用该tp\_call函数，联想到\_\_call\_\_特殊函数，在Python层次，如果一个类有定义\_\_call\_\_特殊函数，
那么最终call这个类时，就会执行这个函数。

接下来会进行Recursive Depth的检测，如果递归调用层次超过了1000（_Py_CheckRecursionLimit）层，就放弃执行

        #define Py_EnterRecursiveCall(where)                                    \
                    (_Py_MakeRecCheck(PyThreadState_GET()->recursion_depth) &&  \
                     _Py_CheckRecursiveCall(where))
        #define Py_LeaveRecursiveCall()                         \
                    (--PyThreadState_GET()->recursion_depth)
        PyAPI_FUNC(int) _Py_CheckRecursiveCall(char *where);
        PyAPI_DATA(int) _Py_CheckRecursionLimit;
        #ifdef USE_STACKCHECK
        #  define _Py_MakeRecCheck(x)  (++(x) > --_Py_CheckRecursionLimit)
        #else
        #  define _Py_MakeRecCheck(x)  (++(x) > _Py_CheckRecursionLimit)
        #endif

## 一个tp_call的实例
我们全文搜索，找到一个tp_call的实例

        PyTypeObject PyType_Type = {
            PyVarObject_HEAD_INIT(&PyType_Type, 0)
            "type",                                     /* tp_name */
            sizeof(PyHeapTypeObject),                   /* tp_basicsize */
            sizeof(PyMemberDef),                        /* tp_itemsize */
            (destructor)type_dealloc,                   /* tp_dealloc */
            0,                                          /* tp_print */
            0,                                          /* tp_getattr */
            0,                                          /* tp_setattr */
            0,                                  /* tp_compare */
            (reprfunc)type_repr,                        /* tp_repr */
            0,                                          /* tp_as_number */
            0,                                          /* tp_as_sequence */
            0,                                          /* tp_as_mapping */
            (hashfunc)_Py_HashPointer,                  /* tp_hash */
            (ternaryfunc)type_call,                     /* tp_call */
            0,                                          /* tp_str */
            (getattrofunc)type_getattro,                /* tp_getattro */
            (setattrofunc)type_setattro,                /* tp_setattro */
            0,                                          /* tp_as_buffer */
            Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
                Py_TPFLAGS_BASETYPE | Py_TPFLAGS_TYPE_SUBCLASS,         /* tp_flags */
            type_doc,                                   /* tp_doc */
            (traverseproc)type_traverse,                /* tp_traverse */
            (inquiry)type_clear,                        /* tp_clear */
            type_richcompare,                                           /* tp_richcompare */
            offsetof(PyTypeObject, tp_weaklist),        /* tp_weaklistoffset */
            0,                                          /* tp_iter */
            0,                                          /* tp_iternext */
            type_methods,                               /* tp_methods */
            type_members,                               /* tp_members */
            type_getsets,                               /* tp_getset */
            0,                                          /* tp_base */
            0,                                          /* tp_dict */
            0,                                          /* tp_descr_get */
            0,                                          /* tp_descr_set */
            offsetof(PyTypeObject, tp_dict),            /* tp_dictoffset */
            type_init,                                  /* tp_init */
            0,                                          /* tp_alloc */
            type_new,                                   /* tp_new */
            PyObject_GC_Del,                            /* tp_free */
            (inquiry)type_is_gc,                        /* tp_is_gc */
        };

这个实例非常特别，因为它是PyType_Type中的实例，也就是类型层次的最顶层，或者说所有类型的Default设置。

        static PyObject *
        type_call(PyTypeObject *type, PyObject *args, PyObject *kwds)
        {
            PyObject *obj;

            if (type->tp_new == NULL) {
                PyErr_Format(PyExc_TypeError,
                             "cannot create '%.100s' instances",
                             type->tp_name);
                return NULL;
            }

            obj = type->tp_new(type, args, kwds);
            if (obj != NULL) {
                /* Ugly exception: when the call was type(something),
                   don't call tp_init on the result. */
                if (type == &PyType_Type &&
                    PyTuple_Check(args) && PyTuple_GET_SIZE(args) == 1 &&
                    (kwds == NULL ||
                     (PyDict_Check(kwds) && PyDict_Size(kwds) == 0)))
                    return obj;
                /* If the returned object is not an instance of type,
                   it won't be initialized. */
                if (!PyType_IsSubtype(obj->ob_type, type))
                    return obj;
                type = obj->ob_type;
                if (PyType_HasFeature(type, Py_TPFLAGS_HAVE_CLASS) &&
                    type->tp_init != NULL &&
                    type->tp_init(obj, args, kwds) < 0) {
                    Py_DECREF(obj);
                    obj = NULL;
                }
            }
            return obj;
        }

可见，如果我们对一个类型对象进行调用时，会使用该类型的tp_new创建一个新的对象并返回，这正是我们创建对象的方式。

        >>> class A(object):
	        def __init__(self):
		        print "Creating A instance"

		
        >>> a = A()
        Creating A instance
        >>> 

因此，对于类型对象的理解，是学习Python源码的核心。
所以接下来，我们继续看一下一些默认的特殊函数实现。

## type_repr的实现
\_\_repr\_\_是用来显示一个能够唯一代表该对象特征的字符串，比如

        >>> repr(a)
        '<__main__.A object at 0x02739870>'
        >>> 

那么它是怎样实现的呢？

        static PyObject *
        type_repr(PyTypeObject *type)
        {
            PyObject *mod, *name, *rtn;
            char *kind;

            mod = type_module(type, NULL);
            if (mod == NULL)
                PyErr_Clear();
            else if (!PyString_Check(mod)) {
                Py_DECREF(mod);
                mod = NULL;
            }
            name = type_name(type, NULL);
            if (name == NULL) {
                Py_XDECREF(mod);
                return NULL;
            }

            if (type->tp_flags & Py_TPFLAGS_HEAPTYPE)
                kind = "class";
            else
                kind = "type";

            if (mod != NULL && strcmp(PyString_AS_STRING(mod), "__builtin__")) {
                rtn = PyString_FromFormat("<%s '%s.%s'>",
                                          kind,
                                          PyString_AS_STRING(mod),
                                          PyString_AS_STRING(name));
            }
            else
                rtn = PyString_FromFormat("<%s '%s'>", kind, type->tp_name);

            Py_XDECREF(mod);
            Py_DECREF(name);
            return rtn;
        }

但是为什么与我们实验得到的结果不一样呢？我们调试一下，首先在typeobject.c/type_call函数中下断点，
然后调试执行刚刚输入的命令，当断点命中后，我们查看一下type类型对象的内容：

        -		type	0x02004418 {_ob_next=0x020023a8 {_ob_next=0x01f890a8 {_ob_next=0x01fe6da8 {_ob_next=0x01fff338 {...} ...} ...} ...} ...}	_typeobject *
        +		_ob_next	0x020023a8 {_ob_next=0x01f890a8 {_ob_next=0x01fe6da8 {_ob_next=0x01fff338 {_ob_next=0x01ffae98 {...} ...} ...} ...} ...}	_object *
        +		_ob_prev	0x01ffe8c0 {_ob_next=0x02004418 {_ob_next=0x020023a8 {_ob_next=0x01f890a8 {_ob_next=0x01fe6da8 {...} ...} ...} ...} ...}	_object *
		        ob_refcnt	0x00000007	int
        +		ob_type	0x1e327df8 {python27_d.dll!_typeobject PyType_Type} {_ob_next=0x1e323594 {python27_d.dll!_object refchain} {...} ...}	_typeobject *
		        ob_size	0x00000000	int
        +		tp_name	0x01f6cfd4 "A"	const char *
		        tp_basicsize	0x00000018	int
		        tp_itemsize	0x00000000	int
		        tp_dealloc	0x1e16eee0 {python27_d.dll!subtype_dealloc(_object *)}	void (_object *) *
		        tp_print	0x00000000	int (_object *, _iobuf *, int) *
		        tp_getattr	0x00000000	_object * (_object *, char *) *
		        tp_setattr	0x00000000	int (_object *, char *, _object *) *
		        tp_compare	0x00000000	int (_object *, _object *) *
		        tp_repr	0x1e1622f0 {python27_d.dll!object_repr(_object *)}	_object * (_object *) *
        +		tp_as_number	0x020044e4 {nb_add=0x00000000 nb_subtract=0x00000000 nb_multiply=0x00000000 ...}	PyNumberMethods *
        +		tp_as_sequence	0x0200458c {sq_length=0x00000000 sq_concat=0x00000000 sq_repeat=0x00000000 ...}	PySequenceMethods *
        +		tp_as_mapping	0x02004580 {mp_length=0x00000000 mp_subscript=0x00000000 mp_ass_subscript=0x00000000 }	PyMappingMethods *
		        tp_hash	0x1e13bb30 {python27_d.dll!_Py_HashPointer(void *)}	long (_object *) *
		        tp_call	0x00000000	_object * (_object *, _object *, _object *) *
		        tp_str	0x1e1629e0 {python27_d.dll!object_str(_object *)}	_object * (_object *) *
		        tp_getattro	0x1e13bf90 {python27_d.dll!PyObject_GenericGetAttr(_object *, _object *)}	_object * (_object *, _object *) *
		        tp_setattro	0x1e13bfb0 {python27_d.dll!PyObject_GenericSetAttr(_object *, _object *, _object *)}	int (_object *, _object *, _object *) *
        +		tp_as_buffer	0x020045b4 {bf_getreadbuffer=0x00000000 bf_getwritebuffer=0x00000000 bf_getsegcount=0x00000000 ...}	PyBufferProcs *
		        tp_flags	0x000e57fb	long
        +		tp_doc	0x00000000 <NULL>	const char *
		        tp_traverse	0x1e16f4c0 {python27_d.dll!subtype_traverse(_object *, int (_object *, void *) *, void *)}	int (_object *, int (_object *, void *) *, void *) *
		        tp_clear	0x1e16eda0 {python27_d.dll!subtype_clear(_object *)}	int (_object *) *
		        tp_richcompare	0x00000000	_object * (_object *, _object *, int) *
		        tp_weaklistoffset	0x00000014	int
		        tp_iter	0x00000000	_object * (_object *) *
		        tp_iternext	0x1e13df60 {python27_d.dll!_PyObject_NextNotImplemented(_object *)}	_object * (_object *) *
        +		tp_methods	0x00000000 <NULL>	PyMethodDef *
        +		tp_members	0x020045d4 {name=0x00000000 <NULL> type=0x00000000 offset=0x00000000 ...}	PyMemberDef *
        +		tp_getset	0x1e327ae8 {python27_d.dll!PyGetSetDef subtype_getsets_full[3]} {name=0x1e20fd88 "__dict__" get=0x1e15e8d0 {python27_d.dll!subtype_dict(_object *, void *)} ...}	PyGetSetDef *
        +		tp_base	0x1e327ec8 {python27_d.dll!_typeobject PyBaseObject_Type} {_ob_next=0x1e327df8 {python27_d.dll!_typeobject PyType_Type} {...} ...}	_typeobject *
        +		tp_dict	0x01ffe8c0 {_ob_next=0x02004418 {_ob_next=0x020023a8 {_ob_next=0x01f890a8 {_ob_next=0x01fe6da8 {...} ...} ...} ...} ...}	_object *
		        tp_descr_get	0x00000000	_object * (_object *, _object *, _object *) *
		        tp_descr_set	0x00000000	int (_object *, _object *, _object *) *
		        tp_dictoffset	0x00000010	int
		        tp_init	0x1e162840 {python27_d.dll!slot_tp_init(_object *, _object *, _object *)}	int (_object *, _object *, _object *) *
		        tp_alloc	0x1e1641b0 {python27_d.dll!PyType_GenericAlloc(_typeobject *, int)}	_object * (_typeobject *, int) *
		        tp_new	0x1e15f5e0 {python27_d.dll!object_new(_typeobject *, _object *, _object *)}	_object * (_typeobject *, _object *, _object *) *
		        tp_free	0x1e0437c0 {python27_d.dll!PyObject_GC_Del(void *)}	void (void *) *
		        tp_is_gc	0x00000000	int (_object *) *
        +		tp_bases	0x01f890a8 {_ob_next=0x01fe6da8 {_ob_next=0x01fff338 {_ob_next=0x01ffae98 {_ob_next=0x01ffadf0 {...} ...} ...} ...} ...}	_object *
        +		tp_mro	0x01ffc738 {_ob_next=0x020051b8 {_ob_next=0x020050f8 {_ob_next=0x01ffe8c0 {_ob_next=0x02004418 {...} ...} ...} ...} ...}	_object *
        +		tp_cache	0x00000000 <NULL>	_object *
        +		tp_subclasses	0x00000000 <NULL>	_object *
        +		tp_weaklist	0x01fe7c98 {_ob_next=0x01ffc738 {_ob_next=0x020051b8 {_ob_next=0x020050f8 {_ob_next=0x01ffe8c0 {...} ...} ...} ...} ...}	_object *
		        tp_del	0x00000000	void (_object *) *
		        tp_version_tag	0x00000060	unsigned int

原来tp\_repr是指向object\_repr的，

        static PyObject *
        object_repr(PyObject *self)
        {
            PyTypeObject *type;
            PyObject *mod, *name, *rtn;

            type = Py_TYPE(self);
            mod = type_module(type, NULL);
            if (mod == NULL)
                PyErr_Clear();
            else if (!PyString_Check(mod)) {
                Py_DECREF(mod);
                mod = NULL;
            }
            name = type_name(type, NULL);
            if (name == NULL) {
                Py_XDECREF(mod);
                return NULL;
            }
            if (mod != NULL && strcmp(PyString_AS_STRING(mod), "__builtin__"))
                rtn = PyString_FromFormat("<%s.%s object at %p>",
                                          PyString_AS_STRING(mod),
                                          PyString_AS_STRING(name),
                                          self);
            else
                rtn = PyString_FromFormat("<%s object at %p>",
                                          type->tp_name, self);
            Py_XDECREF(mod);
            Py_DECREF(name);
            return rtn;
        }

        static PyObject *
        object_str(PyObject *self)
        {
            unaryfunc f;

            f = Py_TYPE(self)->tp_repr;
            if (f == NULL)
                f = object_repr;
            return f(self);
        }

这回与我们的结果是一致的。
那为什么我们自定义的函数没有调用到Default的type类型对象的tp_repr函数呢，
原因是在顶层的类型对象与我们自己定义的类型对象之间某一层次上已经有了实现，这个实现覆盖了顶层的实现，
那么这个实现是什么呢，我们通过全文搜索object\_repr找到了答案:

        PyTypeObject PyBaseObject_Type = {
            PyVarObject_HEAD_INIT(&PyType_Type, 0)
            "object",                                   /* tp_name */
            sizeof(PyObject),                           /* tp_basicsize */
            0,                                          /* tp_itemsize */
            object_dealloc,                             /* tp_dealloc */
            0,                                          /* tp_print */
            0,                                          /* tp_getattr */
            0,                                          /* tp_setattr */
            0,                                          /* tp_compare */
            object_repr,                                /* tp_repr */
            0,                                          /* tp_as_number */
            0,                                          /* tp_as_sequence */
            0,                                          /* tp_as_mapping */
            (hashfunc)_Py_HashPointer,                  /* tp_hash */
            0,                                          /* tp_call */
            object_str,                                 /* tp_str */
            PyObject_GenericGetAttr,                    /* tp_getattro */
            PyObject_GenericSetAttr,                    /* tp_setattro */
            0,                                          /* tp_as_buffer */
            Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE, /* tp_flags */
            PyDoc_STR("The most base type"),            /* tp_doc */
            0,                                          /* tp_traverse */
            0,                                          /* tp_clear */
            0,                                          /* tp_richcompare */
            0,                                          /* tp_weaklistoffset */
            0,                                          /* tp_iter */
            0,                                          /* tp_iternext */
            object_methods,                             /* tp_methods */
            0,                                          /* tp_members */
            object_getsets,                             /* tp_getset */
            0,                                          /* tp_base */
            0,                                          /* tp_dict */
            0,                                          /* tp_descr_get */
            0,                                          /* tp_descr_set */
            0,                                          /* tp_dictoffset */
            object_init,                                /* tp_init */
            PyType_GenericAlloc,                        /* tp_alloc */
            object_new,                                 /* tp_new */
            PyObject_Del,                               /* tp_free */
        };

回想Python的面向对象特性，任何对象都是继承于object对象，因此在object的类型对象中实现了一些公用的函数。
而到了object这一层次还没有实现函数，才会去type类型对象中去查找。

这里插播一条无关内容，我们看PyObject里面都包含了哪些内容，直观的结果如下：

        -		name	0x0040ee28 {_ob_next=0x0040ee00 {_ob_next=0x00416c98 {_ob_next=0x00416c40 {_ob_next=0x00416be8 {...} ...} ...} ...} ...}	_object *
        +		_ob_next	0x0040ee00 {_ob_next=0x00416c98 {_ob_next=0x00416c40 {_ob_next=0x00416be8 {_ob_next=0x00416b90 {...} ...} ...} ...} ...}	_object *
        +		_ob_prev	0x003f02c8 {_ob_next=0x0040ee28 {_ob_next=0x0040ee00 {_ob_next=0x00416c98 {_ob_next=0x00416c40 {...} ...} ...} ...} ...}	_object *
		        ob_refcnt	0x00000001	int
        +		ob_type	0x1e325e88 {python27_d.dll!_typeobject PyString_Type} {_ob_next=0x0036c1a0 {_ob_next=0x003694f8 {_ob_next=...} ...} ...}	_typeobject *

这其实只是头部，包含了双向链表的指针，引用计数，以及比较关键的类型信息；特定于某一类型的信息保存在接下来的区域中。
也正是因为这个原因，我们在调试中看到的大部分内容都只显示出了头部的信息，看不到具体信息。或许我们可以编写一个Visual Studio的插件，
根据type信息来智能判断PyObject的真正类型，然后显示出其真正的信息。

当然，我们也可以手动在Watch中进行类型转换，比如

        -		(PyStringObject*)name	0x003663e0 {_ob_next=0x00369138 {_ob_next=0x003690e8 {_ob_next=0x003690b8 {_ob_next=0x003663a8 {...} ...} ...} ...} ...}	PyStringObject *
        +		_ob_next	0x00369138 {_ob_next=0x003690e8 {_ob_next=0x003690b8 {_ob_next=0x003663a8 {_ob_next=0x00369078 {...} ...} ...} ...} ...}	_object *
        +		_ob_prev	0x00369178 {_ob_next=0x003663e0 {_ob_next=0x00369138 {_ob_next=0x003690e8 {_ob_next=0x003690b8 {...} ...} ...} ...} ...}	_object *
		        ob_refcnt	0x0000003e	int
        +		ob_type	0x1e325e88 {python27_d.dll!_typeobject PyString_Type} {_ob_next=0x0036c1a0 {_ob_next=0x003694f8 {_ob_next=...} ...} ...}	_typeobject *
		        ob_size	0x00000008	int
		        ob_shash	0xa350c618	long
		        ob_sstate	0x00000001	int
        +		ob_sval	0x003663fc "__dict__"	char[0x00000001]

