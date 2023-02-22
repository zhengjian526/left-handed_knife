---
layout: post
title: Interface Libraries
categories: cmake
description: Interface Libraries
keywords: Linux, cmake, compile
---

## Interface Libraries
```cmake
add_library(<name> INTERFACE)
```

创建一个[接口库](https://runebook.dev/zh/docs/cmake/manual/cmake-buildsystem.7#interface-libraries)。一个 `INTERFACE` 库目标不编译源代码，并且不产生磁盘库神器。但是，它可能已设置了属性，并且可能已安装和导出。通常，使用以下命令在接口目标上填充 `INTERFACE_*` 属性：

- [`set_property()`](https://runebook.dev/zh/docs/cmake/command/set_property#command:set_property),
- [`target_link_libraries(INTERFACE)`](https://runebook.dev/zh/docs/cmake/command/target_link_libraries#command:target_link_libraries),
- [`target_link_options(INTERFACE)`](https://runebook.dev/zh/docs/cmake/command/target_link_options#command:target_link_options),
- [`target_include_directories(INTERFACE)`](https://runebook.dev/zh/docs/cmake/command/target_include_directories#command:target_include_directories),
- [`target_compile_options(INTERFACE)`](https://runebook.dev/zh/docs/cmake/command/target_compile_options#command:target_compile_options),
- [`target_compile_definitions(INTERFACE)`](https://runebook.dev/zh/docs/cmake/command/target_compile_definitions#command:target_compile_definitions), and
- [`target_sources(INTERFACE)`](https://runebook.dev/zh/docs/cmake/command/target_sources#command:target_sources),

然后像其他任何目标一样，将其用作[ `target_link_libraries()` ](https://runebook.dev/zh/docs/cmake/command/target_link_libraries#command:target_link_libraries)的参数。

用上述签名创建的接口库本身没有源文件,在生成的构建系统中不作为目标。

版本3.15中的新增功能：接口库可以具有[ `PUBLIC_HEADER` ](https://runebook.dev/zh/docs/cmake/prop_tgt/public_header#prop_tgt:PUBLIC_HEADER)和[ `PRIVATE_HEADER` ](https://runebook.dev/zh/docs/cmake/prop_tgt/private_header#prop_tgt:PRIVATE_HEADER)属性。可以使用[ `install(TARGETS)` ](https://runebook.dev/zh/docs/cmake/command/install#command:install)命令安装这些属性指定的头。

3.19版的新内容:可以用源文件创建一个接口库目标。

```cmake
add_library(<name> INTERFACE [<source>...] [EXCLUDE_FROM_ALL])
```

源文件可以直接在 `add_library` 调用中列出，也可以稍后通过使用 `PRIVATE` 或 `PUBLIC` 关键字调用[ `target_sources()` ](https://runebook.dev/zh/docs/cmake/command/target_sources#command:target_sources)来添加。

如果接口库具有源文件（即，设置了[ `SOURCES` ](https://runebook.dev/zh/docs/cmake/prop_tgt/sources#prop_tgt:SOURCES)target属性），则它将在生成的构建系统中作为构建目标出现，就像[ `add_custom_target()` ](https://runebook.dev/zh/docs/cmake/command/add_custom_target#command:add_custom_target)命令定义的目标一样。它不编译[ `add_custom_command()` ](https://runebook.dev/zh/docs/cmake/command/add_custom_command#command:add_custom_command)，但是包含由add_custom_command（）命令创建的自定义命令的构建规则。

**Note**

在大多数显示 `INTERFACE` 关键字的命令签名中，其后列出的项目仅成为目标使用要求的一部分，而不是目标自身设置的一部分。但是，在 `add_library` 的此签名中， `INTERFACE` 关键字仅引用库类型。在 `add_library` 调用之后列出的源是接口库的 `PRIVATE` ，并且不会出现在其[ `INTERFACE_SOURCES` ](https://runebook.dev/zh/docs/cmake/prop_tgt/interface_sources#prop_tgt:INTERFACE_SOURCES)目标属性中。

## Imported Libraries

```cmake
add_library(<name> <type> IMPORTED [GLOBAL])
```

创建一个名为 `<name>` 的[导入库目标](https://runebook.dev/zh/docs/cmake/manual/cmake-buildsystem.7#imported-targets)。不会生成任何规则来构建它，并且[ `IMPORTED` ](https://runebook.dev/zh/docs/cmake/prop_tgt/imported#prop_tgt:IMPORTED)目标属性为 `True` 。目标名称在创建它的目录及其下的目录中具有作用域，但是 `GLOBAL` 选项扩展了可见性。可以像在项目中构建的任何目标一样引用它。 `IMPORTED` 库对于从诸如[ `target_link_libraries()` 之](https://runebook.dev/zh/docs/cmake/command/target_link_libraries#command:target_link_libraries)类的命令中方便引用非常有用。通过设置名称以 `IMPORTED_` 和 `INTERFACE_` 开头的属性来指定有关导入的库的详细信息。

所述 `<type>` 必须是以下之一：

**STATIC, SHARED, MODULE, UNKNOWN**

引用位于项目外部的库文件。所述[ `IMPORTED_LOCATION` ](https://runebook.dev/zh/docs/cmake/prop_tgt/imported_location#prop_tgt:IMPORTED_LOCATION)目标属性（或它的每构变体 `IMPORTED_LOCATION_<CONFIG>` ）指定在磁盘上的主库文件的位置：

- 对于大多数非 Windows 平台上的 `SHARED` 库，主库文件是链接器和动态加载器使用的 `.so` 或 `.dylib` 文件。如果引用的库文件具有 `SONAME` （或在 macOS 上具有以 `@rpath/` 开头的 `LC_ID_DYLIB` ），则应在[ `IMPORTED_SONAME` ](https://runebook.dev/zh/docs/cmake/prop_tgt/imported_soname#prop_tgt:IMPORTED_SONAME)目标属性中设置该字段的值。如果被引用的库文件不具有 `SONAME` ，但平台支持的话，那么[ `IMPORTED_NO_SONAME` ](https://runebook.dev/zh/docs/cmake/prop_tgt/imported_no_soname#prop_tgt:IMPORTED_NO_SONAME)目标属性应该设置。
- 对于 `SHARED` 在Windows中，库[ `IMPORTED_IMPLIB` ](https://runebook.dev/zh/docs/cmake/prop_tgt/imported_implib#prop_tgt:IMPORTED_IMPLIB)目标属性（或它的每构变体 `IMPORTED_IMPLIB_<CONFIG>` ）指定了DLL的导入库文件（的位置 `.lib` 或 `.dll.a` 在磁盘上），且 `IMPORTED_LOCATION` 是位置所述的 `.dll` 运行时库（和是可选的）。

可以在 `INTERFACE_*` 属性中指定其他使用要求。

一个 `UNKNOWN` 库类型通常只在实现中使用[查找模块](https://runebook.dev/zh/docs/cmake/manual/cmake-developer.7#find-modules)。它允许使用导入的库的路径（通常使用[ `find_library()` ](https://runebook.dev/zh/docs/cmake/command/find_library#command:find_library)命令找到），而不必知道它是什么类型的库。这在Windows中静态库和DLL的导入库都具有相同文件扩展名的Windows上特别有用。

**OBJECT**

引用位于项目外部的一组目标文件。所述[ `IMPORTED_OBJECTS` ](https://runebook.dev/zh/docs/cmake/prop_tgt/imported_objects#prop_tgt:IMPORTED_OBJECTS)目标属性（或它的每构变体 `IMPORTED_OBJECTS_<CONFIG>` ）指定的目标文件的磁盘上的位置。可以在 `INTERFACE_*` 属性中指定其他使用要求。

- `INTERFACE`

  不引用磁盘上的任何库或目标文件，但可以在 `INTERFACE_*` 属性中指定使用要求。

有关更多信息，请参见 `IMPORTED_*` 和 `INTERFACE_*` 属性的文档。

## 参考

- https://runebook.dev/zh/docs/cmake/command/add_library?page=2
