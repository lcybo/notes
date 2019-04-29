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

cppInterpreter对应zero hotspot: 看 `zero page <http://openjdk.java.net/projects/zero/>`_ 和 `build a zero version <https://github.com/unofficial-openjdk/openjdk/blob/jdk/jdk/doc/building.md#libffi>`_

默认的build为templateInterpreter版本

TODO