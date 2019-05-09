**********
JVM解释器
**********

根据abstractInterpreter.hpp中的注释：

 | // This file contains the platform-independent parts
 | // of the abstract interpreter and the abstract interpreter generator.
 | 
 | // Organization of the interpreter(s). There exists two different interpreters in hotpot
 | // an assembly language version (aka template interpreter) and a high level language version
 | // (aka c++ interpreter). Th division of labor is as follows:
 | 
 | // Template Interpreter          C++ Interpreter        Functionality
 | //
 | // templateTable*                bytecodeInterpreter*   actual interpretation of bytecodes
 | //
 | // templateInterpreter*          cppInterpreter*        generation of assembly code that creates
 | //                                                      and manages interpreter runtime frames.
 | //                                                      Also code for populating interpreter
 | //                                                      frames created during deoptimization.
 
jvm源码内存在两种解释器：templateInterpreter和cppInterpreter.

--------------
cppInterpreter
--------------

cppInterpreter对应zero hotspot: 看 `zero page <http://openjdk.java.net/projects/zero/>`_ 和 `build a zero version <https://github.com/unofficial-openjdk/openjdk/blob/jdk/jdk/doc/building.md#libffi>`_

.. code::

  void BytecodeInterpreter::run(interpreterState istate)
  |-void CppInterpreter::main_loop(int recurse, TRAPS)
    |-void maybe_deoptimize(int deoptimized_frames, TRAPS)
      |-void invoke(Method* method, TRAPS)
        |-void CppInterpreter::invoke_method(Method* method, address entry_point, TRAPS)
          // call java from C
          |-void call_stub(JavaCallWrapper *call_wrapper, intptr_t* result, BasicType result_type, Method* method, address entry_point, intptr_t* parameters, int parameter_words, TRAPS)
          // call java from java
          |-void CppInterpreter::main_loop(int recurse, TRAPS)


-------------------
templateInterpreter
-------------------

默认的build为templateInterpreter版本，根据对应平台生成template.

以ubuntu环境为例，首先安装hsdis(和其他资料中的jdk8不同，jdk11的hsdis-amd64.so的安装位置位于%JAVA_HOME/lib/server)，添加VM options: -XX:+UnlockDiagnosticVMOptions -XX:+PrintInterpreter并运行java生成模板: `templates <templates>`_

生成模板的逻辑在templateInterpreterGenerator.cpp中。

.. code::

  void TemplateInterpreterGenerator::generate_all() {
    // slow signature handler
    { CodeletMark cm(_masm, "slow signature handler");
      AbstractInterpreter::_slow_signature_handler = generate_slow_signature_handler();
    }

    // error exits
    { CodeletMark cm(_masm, "error exits");
      _unimplemented_bytecode    = generate_error_exit("unimplemented bytecode");
      _illegal_bytecode_sequence = generate_error_exit("illegal bytecode sequence - method not verified");
    }

    // ... 对应CodeletMark的description

    // 普通java method
    method_entry(zerolocals)
    // 带有synchronized qualifier的java method
    method_entry(zerolocals_synchronized)
    // 下面是编译器识别后优化的entries
    method_entry(empty)
    method_entry(accessor)
    method_entry(abstract)
    method_entry(java_lang_math_sin  )
    method_entry(java_lang_math_cos  )
    method_entry(java_lang_math_tan  )
    method_entry(java_lang_math_abs  )
    method_entry(java_lang_math_sqrt )
    method_entry(java_lang_math_log  )
    method_entry(java_lang_math_log10)
    method_entry(java_lang_math_exp  )
    method_entry(java_lang_math_pow  )
    method_entry(java_lang_math_fmaF )
    method_entry(java_lang_math_fmaD )
    method_entry(java_lang_ref_reference_get)
    method_entry(java_util_zip_CRC32_update)
    method_entry(java_util_zip_CRC32_updateBytes)
    method_entry(java_util_zip_CRC32_updateByteBuffer)
    method_entry(java_util_zip_CRC32C_updateBytes)
    method_entry(java_util_zip_CRC32C_updateDirectByteBuffer)
    method_entry(java_lang_Float_intBitsToFloat);
    method_entry(java_lang_Float_floatToRawIntBits);
    method_entry(java_lang_Double_longBitsToDouble);
    method_entry(java_lang_Double_doubleToRawLongBits);
    // native method
    method_entry(native)
    // synchronized native method
    method_entry(native_synchronized)
    // 
    set_entry_points_for_all_bytes();
  }



