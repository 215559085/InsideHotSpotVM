# [Inside HotSpot] 加载字节码到JVM

## 1. 字节码文件解析器
[Java虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se12/html/index.html)有提到，一个Java类要想被JVM使用必须经过`加载->链接->初始化`的步骤。字节码解析器就是Java类进入JVM的第一个门槛，它负责将有序的二进制字节码文件(`.class`)解析成JVM内部需要的形式，说直白点就是结构体，JVM创建结构体，并用二进制文件的内容填充结构体，这即是第一步**加载**的工作。
和加载有关的大部分代码位于`hotspot/share/classfile`目录。ClassFileParser::parse_stream会负责最主要的解析流程(为了不影响流程，里面的断言和异常处理已经去除)：
```cpp
// hotspot\share\classfile\classFileParser.cpp
void ClassFileParser::parse_stream(const ClassFileStream* const stream,
                                   TRAPS) {
  // 魔数0xfacebabe
  const u4 magic = stream->get_u4_fast();
  // minor major版本号
  _minor_version = stream->get_u2_fast();
  _major_version = stream->get_u2_fast();
  // 常量池
  ...
}
```
ClassFileStream简单封装了一下文件读取的接口，创建了一个缓冲区。然后parse_stream()解析流程请完全对照着[Java虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se12/html/jvms-4.html)的class文件结构来，这个我也讲不出花来...:
```cpp
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
每个字段在文档中都有非常详细的解释，简单来说：

+ `magic` 标识这是字节码文件，是一个常量(0xcafebabe)
+ `minor_version` class文件最小版本号
+ `major_version` class文件最大版本号
+ `constant_pool_count` 常量池大小
+ `constant_pool[]` 常量池
+ `access_flags` 类访问修饰符，比如public，final
+ `this_class` 当前类在常量池的索引号
+ `super_class` 父类在常量池的索引号，如果父类是java.lang.Object则是0
+ `interfaces_count` 接口个数
+ `interfaces[]` 接口
+ `fields_count` 字段个数
+ `fields[]` 字段
+ `methods_count` 方法个数
+ `methods[]` 方法
+ `attributes_count` 属性个数
+ `attributes[]` 属性，比如源码文件名，内部类名，一些辅助信息。

但是难得hotspot里面有一个模块是如此的简单，我们可以再仔细研究研究看看能不能有意外收获。

## 2. 类加载器
### 2.1 Bootstrap类加载器
第一部分提到类解析器会将字节码文件解析成JVM内部用到的结构体，如果了解过oop-klass模型的朋友一定知道，这个结构体就是klass，或者更具体一点是**InstanceKlass**。沟通字节码文件和InstanceKlass的桥梁就是**klassFactory**。klass工厂会调用ClassFileParser将读到的字节码转化为InstanceKlass作为工厂的产出，而工厂的调用者就是著名的JVM类加载器，即Bootstrap类加载器，它位于`classfile/classloader`：
```cpp
// hotspot\share\classfile\classLoader.cpp
class ClassLoader: AllStatic {
 public:
  enum ClassLoaderType {
    BOOT_LOADER = 1,      /* boot loader */
    PLATFORM_LOADER  = 2, /* PlatformClassLoader */
    APP_LOADER  = 3       /* AppClassLoader */
  };
 protected:
 	...
 	// 类加载的的最终落地
	InstanceKlass* ClassLoader::load_class(Symbol* name, bool search_append_only, TRAPS) {
	  ResourceMark rm(THREAD);
	  HandleMark hm(THREAD);

      // 类名字，注意在虚拟机中类的包的那个点会用斜杠代替
      // 比如java.lang.String在虚拟机中是java/lang/Class
	  const char* const class_name = name->as_C_string();
	  EventMark m("loading class %s", class_name);
      // 文件名字，java/lang/String.class
	  const char* const file_name = file_name_for_class_name(class_name,
	                                                         name->utf8_length());
	  ClassFileStream* stream = NULL;
	  s2 classpath_index = 0;
	  ClassPathEntry* e = NULL;

	  // 尝试加载#1 [--patch-module]
	  if (_patch_mod_entries != NULL && !search_append_only) {
	    if (!DumpSharedSpaces) {
	      stream = search_module_entries(_patch_mod_entries, class_name, 
	      	file_name, CHECK_NULL);
	    }
	  }

	  // 尝试加载#2 [jimage | exploded build]
	  if (!search_append_only && (NULL == stream)) {
	    if (has_jrt_entry()) {
	      e = _jrt_entry;
	      stream = _jrt_entry->open_stream(file_name, CHECK_NULL);
	    } else {
	      // Exploded build - attempt to locate class in its defining module's location.
	      assert(_exploded_entries != NULL, "No exploded build entries present");
	      stream = search_module_entries(_exploded_entries, class_name, file_name, CHECK_NULL);
	    }
	  }

	  // 尝试加载#3 [-Xbootclasspath/a | jvmti appended entries]
	  if (search_append_only && (NULL == stream)) {
	    classpath_index = 1;
	    e = _first_append_entry;
	    while (e != NULL) {
	      stream = e->open_stream(file_name, CHECK_NULL);
	      if (NULL != stream) {break;}
	      e = e->next();
	      ++classpath_index;
	    }
	  }
	  if (NULL == stream) {return NULL;}

	  stream->set_verify(ClassLoaderExt::should_verify(classpath_index));
	  ClassLoaderData* loader_data = ClassLoaderData::the_null_class_loader_data();
	  Handle protection_domain;

	  // 从加载到的字节码中产出InstanceKlass
	  InstanceKlass* result = 
	  KlassFactory::create_from_stream(stream,name,loader_data,protection_domain,
	                                   NULL, NULL,THREAD);
	  // 加入包
	  if (!add_package(file_name, classpath_index, THREAD)) {
	    return NULL;
	  }
	  return result;
	}
}
```
加载完类之后add_package会把该类加入包(PackageEntryTable)，包是一个哈希表的结构：
```BASH
| 包A | ->[类A,类B...]
| 包B | ->[类C,类D...]
```