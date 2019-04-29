************************
declaration(amd64/linux)
************************

CONSTANT_REGISTER_DECLARATION

.. code::

    // example
    CONSTANT_REGISTER_DECLARATION(Register, G0, 0);
    extern const Register G0 ;
    enum { G0_RegisterEnumValue = 0 } ;
    // declarations
    CONSTANT_REGISTER_DECLARATION(Register, rax,    (0));
    CONSTANT_REGISTER_DECLARATION(Register, rcx,    (1));
    CONSTANT_REGISTER_DECLARATION(Register, rdx,    (2));
    CONSTANT_REGISTER_DECLARATION(Register, rbx,    (3));
    CONSTANT_REGISTER_DECLARATION(Register, rsp,    (4));
    CONSTANT_REGISTER_DECLARATION(Register, rbp,    (5));
    CONSTANT_REGISTER_DECLARATION(Register, rsi,    (6));
    CONSTANT_REGISTER_DECLARATION(Register, rdi,    (7));
    CONSTANT_REGISTER_DECLARATION(Register, r8,     (8));
    CONSTANT_REGISTER_DECLARATION(Register, r9,     (9));
    CONSTANT_REGISTER_DECLARATION(Register, r10,   (10));
    CONSTANT_REGISTER_DECLARATION(Register, r11,   (11));
    CONSTANT_REGISTER_DECLARATION(Register, r12,   (12));
    CONSTANT_REGISTER_DECLARATION(Register, r13,   (13));
    CONSTANT_REGISTER_DECLARATION(Register, r14,   (14));
    CONSTANT_REGISTER_DECLARATION(Register, r15,   (15));


REGISTER_DECLARATION

::

|-------------------------------------------------------|
| c_rarg0   c_rarg1  c_rarg2 c_rarg3 c_rarg4 c_rarg5    |
|-------------------------------------------------------|
| rcx       rdx      r8      r9      rdi*    rsi*       | windows (* not a c_rarg)
| rdi       rsi      rdx     rcx     r8      r9         | solaris/linux
|-------------------------------------------------------|
| j_rarg5   j_rarg0  j_rarg1 j_rarg2 j_rarg3 j_rarg4    |
|-------------------------------------------------------|

**c stand for c language, j stand for java**

.. code::

    //example
    REGISTER_DECLARATION(Register, Gmethod, G5);
    extern const Register Gmethod ;
    enum { Gmethod_RegisterEnumValue = G5_RegisterEnumValue } ;
    // declarations
    REGISTER_DECLARATION(Register, c_rarg0, rdi);
    REGISTER_DECLARATION(Register, c_rarg1, rsi);
    REGISTER_DECLARATION(Register, c_rarg2, rdx);
    REGISTER_DECLARATION(Register, c_rarg3, rcx);
    REGISTER_DECLARATION(Register, c_rarg4, r8);
    REGISTER_DECLARATION(Register, c_rarg5, r9);
    REGISTER_DECLARATION(XMMRegister, c_farg0, xmm0);
    REGISTER_DECLARATION(XMMRegister, c_farg1, xmm1);
    REGISTER_DECLARATION(XMMRegister, c_farg2, xmm2);
    REGISTER_DECLARATION(XMMRegister, c_farg3, xmm3);
    REGISTER_DECLARATION(XMMRegister, c_farg4, xmm4);
    REGISTER_DECLARATION(XMMRegister, c_farg5, xmm5);
    REGISTER_DECLARATION(XMMRegister, c_farg6, xmm6);
    REGISTER_DECLARATION(XMMRegister, c_farg7, xmm7);
    REGISTER_DECLARATION(Register, j_rarg0, c_rarg1);
    REGISTER_DECLARATION(Register, j_rarg1, c_rarg2);
    REGISTER_DECLARATION(Register, j_rarg2, c_rarg3);
    REGISTER_DECLARATION(Register, j_rarg3, c_rarg4);
    REGISTER_DECLARATION(Register, j_rarg4, c_rarg5);
    REGISTER_DECLARATION(Register, j_rarg5, c_rarg0);

