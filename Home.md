Welcome to the pyarmor wiki!

Pyarmor is an obfuscator of Python scripts. It used to obfuscate python scripts.

# How to Obfuscate Python Script by Pyarmor

From Pyarmor 3.3, a new mode is introduced. By this way, no import
hooker, no setprofile, no settrace. The performance of running or
importing obfuscation python script has been remarkably improved.

Pyarmor protects Python scripts by the following ways:

* Obfuscate code object to protect constants and literal strings.
* Obfuscate byte code of each code object.
* Clear f_locals of frame as soon as code object executation completed.

There are 2 different cases for Pyarmor to protect Python scripts:

* Application, also called standalone package
* Package used by others

In the first case, Pyarmor obfuscates all the Python Scripts belong to
standalone application, and doesn't allow to import those obfuscated
scripts from any other clear script. So a simple way can be apply, it
called **Restrict Mode**.

For the second case, things get a little complicated. Any other script
can import these obfuscated scripts, the frame and code object can be
accessed in outer. It need more work to protect Python scripts.

## Mechanism in Restrict Mode

In restrict mode, Pyarmor will restore obfuscated byte code when this
code object is called first time, and not obfuscate it again. It's
efficient and enough, because code object can't be accessed from any
other scripts, except obfuscated scripts.

### Obfuscate Python Scripts

- Compile python source file to code object

```
    char * filename = "xxx.py";
    char * source = read_file( filename );
    PyObject *co = Py_CompileString( source, filename, Py_file_input );
```

- Iterate code object, wrap bytecode of each code object as the
  following format

```
    0   JUMP_ABSOLUTE            n = 3 + len(bytecode)

    3
    ...
    ... Here it's obfuscated bytecode
    ...

    n   LOAD_GLOBAL              ? (__armor__)
    n+3 CALL_FUNCTION            0
    n+6 POP_TOP
    n+7 JUMP_ABSOLUTE            0

```

- Serialize code object and obfuscate it

```
    char *original_code = marshal.dumps( co );
    char *obfuscated_code = obfuscate_algorithm( original_code  );
```

- Create wrapper script "xxx.py", **${obfuscated_code}** stands for string constant generated in previous step.

```
    __pyarmor__(__name__, b'${obfuscated_code}')
```

### Run or Import Obfuscated Python Scripts

When import or run this wrapper script, the first statement is to call a CFunction:

```
    int __pyarmor__(char *name, unsigned char *obfuscated_code) {
      char *original_code = resotre_obfuscated_code( obfuscated_code );
      PyObject *co = marshal.loads( original_code );
      PyObject *mod = PyImport_ExecCodeModule( name, co );
    }
```

This function accepts 2 parameters: module name and obfuscated code, then

* Restore obfuscated code
* Create a code object by original code
* Import original module **(this will result in a duplicated frame in
  traceback)**

#### Run or Import Obfuscated Bytecode

After module imported, when any code object in this module is called
first time, from the wrapped bytecode descripted in above section, we
know

- First op is JUMP_ABSOLUTE, it will jump to offset n

- At offset n, the instruction is to call a PyCFunction. This function
  will restore those obfuscated bytecode between offset 3 and n, and
  place the original bytecode at offset 0

- After function call, the last instruction is to jump to
  offset 0. The really bytecode now is executed.

### Implementation

* Use Command `obfuscate`

```
    # Obfuscated scripts are saved in default output path "dist"
    python pyarmor.py obfuscate --src=examples/simple \
                                --entry=queens.py "*.py"
    cd dist
    cat queens.py

    # Run obfuscated script
    python queens.py

```

* Use command `init` to create project configured as application


```
    mkdir projects
    python pyarmor.py init --type=app --src=/PATH/TO/SCRIPTS \
                           --entry=main.py projects/myapp
    cd projects/myapp

    # Now obfuscate scripts
    cd projects/myapp
    ./pyarmor build

    # Run obfuscated scripts
    cd dist
    python main.py

```

Use command `config` to configure other obfuscate modes:

```
    # First create a project to manage obfuscated scripts
    python pyarmor.py init --src=/PATH/TO/SCRIPTS projects/myproject
    cd projects/myproject

    # Second, use command "config" to specify obfuscation mode:
    #
    #    --obf-module-mode 'des' is default. Obfuscate module by DES
    #                      'none' means no obfuscate module object.
    #
    #    --obf-code-mode   'des' is default.
    #                      'fast' is a simple algorithm faster than DES.
    #                      'none' means no obfuscate bytecode.
    #
    # For example (in windows, run ./pyarmor.bat other than ./pyarmor),
    ./pyarmor config --obf-module-mode des
                     --obf-code-mode none

    # Finally, obfuscate scripts as project configuration by command 'build'
    ./pyarmor build

```

## Mechanism Without Restrict Mode

This feature is introuced from v3.9.0

When restrict mode is disabled, it means obfuscated scripts can be
imported from any other scripts. So every code object must be
obfuscated again as soon as it returns. Pyarmor insert a
`try...finally` block in each code object, it will modify each code
object as the following way:

* Add wrap header to call `__armor_enter__` before run this code object

```
    LOAD_GLOBALS    N (__armor_enter__)
    CALL_FUNCTION   0
    POP_TOP
    SETUP_FINALLY   X (jump to wrap footer)

```

* Following the header it's original byte-code.

* The oparg of each absolute jump instruction is increased by size of wrap header

* Obfuscate the original byte-code

* Append the wrap footer to call `__armor_exit__`

```
    LOAD_GLOBALS    N + 1 (__armor_exit__)
    CALL_FUNCTION   0
    POP_TOP
    END_FINALLY

```

* The `co->stacksize` of code object is increased by 2

When code object is executed, `__armor_enter__` will restore original
byte-code first. Before it returns, call `__armor_exit__` to obfuscate
original byte-code again. Besides, `__armor_exit__` will clear all the
locals in this frame.

### Implementation

From Pyarmor 3.9.0, there are 2 ways

* Use Command `obfuscate` with option `--no-restrict`

```
    # Here is a simple case, show how to import obfuscated module
    # 'queens.py' from clear script 'hello.py'

    # Obfuscate module with command 'obfuscate'
    # The key is option no-restrict
    python pyarmor.py obfuscate --no-restrict \
                                --src=examples/simple \
                                --entry=hello.py \
                                queens.py

    # After this command:
    #
    # The runtime files are in the output path 'dist'
    # Bootstrap code is inserted into hello.py', saved in 'dist'
    # The obfuscated module 'queens.py' is saved in 'dist'

    # Now run hello.py to import obfuscated module 'queens'
    cd dist
    python hello.py

```

* Use command `init` to create project configured as package


```
    # Here is a typical case, show how clear script 'main.py'
    # imports obfuscated package 'mypkg'

    # First create a project, configure as package
    python pyarmor.py init --type=package \
                           --src=/PATH/TO/mypkg \
                           --entry=/ABSOLUTE/PATH/TO/main.py \
                           projects/testpkg

    # Now obfuscate scripts
    cd projects/mypkg
    ./pyarmor build

    # After build:
    #
    # The runtime files are in the output path 'dist'
    # Bootstrap code is inserted into 'main.py', and saved in 'dist'
    # The obfuscated scripts of package are in the subpath 'dist/mypkg'

    # Run main.py to import obfuscated package 'mypkg'
    cd dist
    python main.py

```

## Performance Analaysis

With default configuration, Pyarmor will do the following extra work
to run or import obfuscated bytecode:

- Load library _pytransform at startup
- Initialize library _pytransform at startup
- Veirfy license at startup
- Restore obfuscated code object of python module
- Restore obfuscated bytecode when code object is called first time

There is command "benchmark" used to run benchmark test

```
    usage: pyarmor.py benchmark [-h] [--obf-module-mode {none,des}]
                                [--obf-code-mode {none,des,fast}]

    optional arguments:
      -h, --help            show this help message and exit
      -m, --obf-module-mode {none,des}
      -c, --obf-code-mode {none,des,fast,wrap}
```

For example, run test in default mode

```
    python pyarmor.py benchmark

    INFO     Start benchmark test ...
    INFO     Obfuscate module mode: des
    INFO     Obfuscate bytecode mode: des
    INFO     Benchmark bootstrap ...
    INFO     Benchmark bootstrap OK.
    INFO     Run benchmark test ...
    load_pytransform: 6.35248334635 ms
    init_pytransform: 3.85942906151 ms
    verify_license: 0.730260410192 ms

    Test script: bfoo.py
    Obfuscated script: obfoo.py
    Start test with mode 8
    --------------------------------------

    import_no_obfuscated_module: 10.3613727443 ms
    import_obfuscated_module: 8.09683912341 ms

    run_empty_no_obfuscated_code_object: 0.00502857206712 ms
    run_empty_obfuscated_code_object: 0.0433015928002 ms

    run_one_thousand_no_obfuscated_bytecode: 0.00446984183744 ms
    run_one_thousand_obfuscated_bytecode: 0.11426033197 ms

    run_ten_thousand_no_obfuscated_bytecode: 0.00474920695228 ms
    run_ten_thousand_obfuscated_bytecode: 0.72383501255 ms
    INFO     Finish benchmark test.

```

Here it's a normal license checked by verify_license. If the license
is bind to fixed machine, for example, mac address, it need more time
to read hardware information.

import_obfuscated_module will first restore obfuscated code object,
then import this pre-compiled code object.

The bytecode size of function one_thousand is about 1K, and
ten_thousand is about 10K. Most of them will not be executed, because
they're in False condition for ever. So run_empty, run_one_thousand,
run_ten_thousand are almost same in non-obfuscated mode, about 0.004
ms.

In obfuscated mode, it's about 0.1 ms for 1K bytecoe, and 0.7~0.8 ms
for 10K bytecode in my laptop. It's mainly consumed by restoring
obfuscated bytecodes.

See another mode, only module obfuscated

```
    python pyarmor.py benchmark --obf-code-mode=none
    INFO     Start benchmark test ...
    INFO     Obfuscate module mode: des
    INFO     Obfuscate bytecode mode: none
    INFO     Benchmark bootstrap ...
    INFO     Benchmark bootstrap OK.
    INFO     Run benchmark test ...
    load_pytransform: 7.96721371012 ms
    init_pytransform: 3.8571941406 ms
    verify_license: 0.728025489273 ms

    Test script: bfoo.py
    Obfuscated script: obfoo.py
    Start test with mode 7
    --------------------------------------

    import_no_obfuscated_module: 10.7399124749 ms
    import_obfuscated_module: 8.19601373918 ms

    run_empty_no_obfuscated_code_object: 0.00530793718196 ms
    run_empty_obfuscated_code_object: 0.00391111160776 ms

    run_one_thousand_no_obfuscated_bytecode: 0.00446984183744 ms
    run_one_thousand_obfuscated_bytecode: 0.00391111160776 ms

    run_ten_thousand_no_obfuscated_bytecode: 0.00446984183744 ms
    run_ten_thousand_obfuscated_bytecode: 0.00391111160776 ms
    INFO     Finish benchmark test.

```
It's even faster than no obfuscated scripts!
