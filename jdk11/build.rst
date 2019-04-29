****************************
Build command for openjdk-11
****************************

Commands
========

1. Clone git repository::

	hg clone http://hg.openjdk.java.net/jdk/jdk
	
2. Checkout to latest update::

	hg up jdk-11+28
	
3. Make sure you have installed valid boot jdk.

4. To build fast debug version jdk, run configure with following options::

        bash configure --enable-debug --with-jvm-variants=server --enable-dtrace --with-jobs=4 --disable-warnings-as-errors

5. make


Reference
=========

`building guide <https://github.com/unofficial-openjdk/openjdk/blob/jdk/jdk/doc/building.md>`_