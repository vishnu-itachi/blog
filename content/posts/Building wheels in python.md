+++
weight = 0
title = 'Building Wheels in Python'
date = 2023-12-21T16:36:28+05:30
draft = false
tags = ['python', 'wheel', 'pypi', 'podman']
image = "/images/building_wheels_in_python/python_wheel.png"
+++

__The other day an office colleague presented me with a problem.__

He was deploying a Python app whose one of the dependency was [Gssapi](https://pypi.org/project/gssapi/). He wanted to deploy onto a rhel8 server which had only Python installed.

Now for that, we need to give all dependencies in a single folder. Surprisingly this [Gssapi](https://pypi.org/project/gssapi/#files) package Linux wheel was not present in the pypi. There is a reason why a linux wheel is not possible. You can read about that here: [python-gssapi/issues/200](https://github.com/pythongssapi/python-gssapi/issues/200).

However, in my case, I will build a wheel to be used in the same linux rhel server.
This becomes more like an installation rather than a generic wheel creation.

**_NOTE:_** If you want to understand what is a _python wheel_ and how python installation works, check out this post by [Real Python](https://realpython.com/python-wheels/#:~:text=Wheels%20install%20faster%20than%20source,of%20the%20corresponding%20source%20distribution.)

### Setup

To emulate the installation process we will use [podman](https://podman.io) (which is a docker alternative). I will be using a Windows machine though it doesn't matter what you use.
We will be using python3.8 for demonstration.

### Problem

Let's try to reproduce the issue which includes the following steps.

1. Create a rhel8 container

```bash
podman run -it rhel8
```

![Rhel8](/images/building_wheels_in_python/rhel8.png)

2. Install python3.8 and pip etc.

```bash
yum install python38.x86_64
```

![Python_install](/images/building_wheels_in_python/python_install.png)

3. Try to install gssapi

```bash
/bin/python3.8 -m pip install gssapi
```

**_NOTE:_** : One should always install packages in a [virtual environment](https://docs.python.org/3/library/venv.html). But here we skip that because well thats not the point here 😄.

```bash
[root@9aebcf0b8195 /]# /bin/python3.8 -m pip install gssapi
WARNING: Running pip install with root privileges is generally not a good idea. Try `python3.8 -m pip install --user` instead.
Collecting gssapi
  Downloading https://files.pythonhosted.org/packages/13/e7/dd88180cfcf243be62308707cc2f5dae4c726c68f30b9367931c794fda16/gssapi-1.8.3.tar.gz (94kB)
     |████████████████████████████████| 102kB 1.1MB/s
  Installing build dependencies ... done
  Getting requirements to build wheel ... error
  ERROR: Command errored out with exit status 1:
   command: /bin/python3.8 /usr/lib/python3.8/site-packages/pip/_vendor/pep517/_in_process.py get_requires_for_build_wheel /tmp/tmpdpbojz58
       cwd: /tmp/pip-install-5b2yckx_/gssapi
  Complete output (21 lines):
  /bin/sh: krb5-config: command not found
  Traceback (most recent call last):
    File "/usr/lib/python3.8/site-packages/pip/_vendor/pep517/_in_process.py", line 257, in <module>
      main()
    File "/usr/lib/python3.8/site-packages/pip/_vendor/pep517/_in_process.py", line 240, in main
      json_out['return_val'] = hook(**hook_input['kwargs'])
    File "/usr/lib/python3.8/site-packages/pip/_vendor/pep517/_in_process.py", line 91, in get_requires_for_build_wheel
      return hook(config_settings)
    File "/tmp/pip-build-env-_94zkxdt/overlay/lib/python3.8/site-packages/setuptools/build_meta.py", line 325, in get_requires_for_build_wheel
      return self._get_build_requires(config_settings, requirements=['wheel'])
    File "/tmp/pip-build-env-_94zkxdt/overlay/lib/python3.8/site-packages/setuptools/build_meta.py", line 295, in _get_build_requires
      self.run_setup()
    File "/tmp/pip-build-env-_94zkxdt/overlay/lib/python3.8/site-packages/setuptools/build_meta.py", line 311, in run_setup
      exec(code, locals())
    File "<string>", line 109, in <module>
    File "<string>", line 22, in get_output
    File "/usr/lib64/python3.8/subprocess.py", line 415, in check_output
      return run(*popenargs, stdout=PIPE, timeout=timeout, check=True,
    File "/usr/lib64/python3.8/subprocess.py", line 516, in run
      raise CalledProcessError(retcode, process.args,
  subprocess.CalledProcessError: Command 'krb5-config --libs gssapi' returned non-zero exit status 127.
  ----------------------------------------
ERROR: Command errored out with exit status 1: /bin/python3.8 /usr/lib/python3.8/site-packages/pip/_vendor/pep517/_in_process.py get_requires_for_build_wheel /tmp/tmpdpbojz58 Check the logs for full command output.
[root@9aebcf0b8195 /]#
```

__OOPSIE DOOPSE AND HERE WE GET THE ERROR 😱 😱__

Let's dig a bit into the error.

### Exploration

Let's just try to run the command shown in the error output.

```bash
[root@b8e29bb9e124 /]# krb5-config --libs gssapi
bash: krb5-config: command not found
```

After a bit of searching ([well the 1st link in Google 😸](https://stackoverflow.com/questions/74854623/gssapi-docker-installation-issue-bin-sh-1-krb5-config-not-found)), we get to know that we need to install krb5 dev.

```bash
yum install krb5-devel.x86_64
```

![Krb5 Install](/images/building_wheels_in_python/krb5_install.png)

okay now let's try running the same command.

```bash
[root@b8e29bb9e124 /]# krb5-config --libs gssapi
-lgssapi_krb5 -lkrb5 -lk5crypto -lcom_err
[root@b8e29bb9e124 /]#
```

Yaay!! 😸😸 we didn't get any error. Now let's try installing it again.

```bash
[root@b8e29bb9e124 /]# /bin/python3.8 -m pip install gssapi
WARNING: Running pip install with root privileges is generally not a good idea. Try `python3.8 -m pip install --user` instead.
Collecting gssapi
  Downloading https://files.pythonhosted.org/packages/13/e7/dd88180cfcf243be62308707cc2f5dae4c726c68f30b9367931c794fda16/gssapi-1.8.3.tar.gz (94kB)
     |████████████████████████████████| 102kB 1.1MB/s
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Installing backend dependencies ... done
    Preparing wheel metadata ... done
Collecting decorator
  Downloading https://files.pythonhosted.org/packages/d5/50/83c593b07763e1161326b3b8c6686f0f4b0f24d5526546bee538c89837d6/decorator-5.1.1-py3-none-any.whl
Building wheels for collected packages: gssapi
  Building wheel for gssapi (PEP 517) ... error
  ERROR: Command errored out with exit status 1:
   command: /bin/python3.8 /usr/lib/python3.8/site-packages/pip/_vendor/pep517/_in_process.py build_wheel /tmp/tmpv6h30g1h
       cwd: /tmp/pip-install-rtvchiss/gssapi
  Complete output (58 lines):
  running bdist_wheel
  running build
  running build_py
  creating build
  creating build/lib.linux-x86_64-cpython-38
  creating build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/names.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/creds.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/sec_contexts.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/mechs.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/exceptions.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/_utils.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/_win_config.py -> build/lib.linux-x86_64-cpython-38/gssapi
  creating build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/named_tuples.py -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  creating build/lib.linux-x86_64-cpython-38/gssapi/raw/_enum_extensions
  copying gssapi/raw/_enum_extensions/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi/raw/_enum_extensions
  creating build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/tests/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/tests/test_raw.py -> build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/tests/test_high_level.py -> build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/py.typed -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/raw/ext_iov_mic.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/names.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc6680_comp_oid.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc6680.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_krb5.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc5588.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_password.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/creds.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_dce.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc5587.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_cred_store.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/oids.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_cred_imp_exp.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_password_add.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/exceptions.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/types.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/message.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_s4u.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_dce_aead.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/sec_contexts.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/mech_krb5.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/chan_bindings.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc4178.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_ggf.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc5801.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/misc.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_set_cred_opt.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  running build_ext
  building 'gssapi.raw.misc' extension
  creating build/temp.linux-x86_64-cpython-38
  creating build/temp.linux-x86_64-cpython-38/gssapi
  creating build/temp.linux-x86_64-cpython-38/gssapi/raw
  gcc -pthread -Wno-unused-result -Wsign-compare -DDYNAMIC_ANNOTATIONS_ENABLED=1 -DNDEBUG -O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -D_GNU_SOURCE -fPIC -fwrapv -O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -D_GNU_SOURCE -fPIC -fwrapv -O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -D_GNU_SOURCE -fPIC -fwrapv -fPIC -Igssapi/raw -I/usr/include/python3.8 -c gssapi/raw/misc.c -o build/temp.linux-x86_64-cpython-38/gssapi/raw/misc.o -DHAS_GSSAPI_EXT_H
  error: command 'gcc' failed: No such file or directory
  ----------------------------------------
  ERROR: Failed building wheel for gssapi
  Running setup.py clean for gssapi
Failed to build gssapi
ERROR: Could not build wheels for gssapi which use PEP 517 and cannot be installed directly
[root@b8e29bb9e124 /]#
```

We again get an error but this time it's a different error. We are making progress!!!

This error is a bit easier to tackle since we need to install [GCC](https://gcc.gnu.org/) the OG C compiler (well it compiles many languages but that's not the point of this article anyhow).

_Why do we need to install GCC for Python you may ask_
  
Well, we are using a cpython interpreter and if we need to build any packages that depend on the cpython APIs (what I mean here loosely is some packages will have .c files that need to be compiled) we need to install GCC.

So let's install gcc.

```bash
[root@b8e29bb9e124 /]# yum install gcc.x86_64
...

[root@b8e29bb9e124 /]# gcc --version
gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-20)
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

```

Voila GCC is now installed. Now let's give it a try again.

```bash
[root@b8e29bb9e124 /]# /bin/python3.8 -m pip install gssapi
WARNING: Running pip install with root privileges is generally not a good idea. Try `python3.8 -m pip install --user` instead.
Collecting gssapi
  Using cached https://files.pythonhosted.org/packages/13/e7/dd88180cfcf243be62308707cc2f5dae4c726c68f30b9367931c794fda16/gssapi-1.8.3.tar.gz
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Installing backend dependencies ... done
    Preparing wheel metadata ... done
Collecting decorator
  Using cached https://files.pythonhosted.org/packages/d5/50/83c593b07763e1161326b3b8c6686f0f4b0f24d5526546bee538c89837d6/decorator-5.1.1-py3-none-any.whl
Building wheels for collected packages: gssapi
  Building wheel for gssapi (PEP 517) ... error
  ERROR: Command errored out with exit status 1:
   command: /bin/python3.8 /usr/lib/python3.8/site-packages/pip/_vendor/pep517/_in_process.py build_wheel /tmp/tmpi8fxoe_0
       cwd: /tmp/pip-install-f1irjrfo/gssapi
  Complete output (62 lines):
  running bdist_wheel
  running build
  running build_py
  creating build
  creating build/lib.linux-x86_64-cpython-38
  creating build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/names.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/creds.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/sec_contexts.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/mechs.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/exceptions.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/_utils.py -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/_win_config.py -> build/lib.linux-x86_64-cpython-38/gssapi
  creating build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/named_tuples.py -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  creating build/lib.linux-x86_64-cpython-38/gssapi/raw/_enum_extensions
  copying gssapi/raw/_enum_extensions/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi/raw/_enum_extensions
  creating build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/tests/__init__.py -> build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/tests/test_raw.py -> build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/tests/test_high_level.py -> build/lib.linux-x86_64-cpython-38/gssapi/tests
  copying gssapi/py.typed -> build/lib.linux-x86_64-cpython-38/gssapi
  copying gssapi/raw/ext_iov_mic.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/names.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc6680_comp_oid.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc6680.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_krb5.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc5588.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_password.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/creds.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_dce.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc5587.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_cred_store.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/oids.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_cred_imp_exp.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_password_add.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/exceptions.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/types.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/message.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_s4u.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_dce_aead.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/sec_contexts.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/mech_krb5.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/chan_bindings.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc4178.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_ggf.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_rfc5801.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/misc.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  copying gssapi/raw/ext_set_cred_opt.pyi -> build/lib.linux-x86_64-cpython-38/gssapi/raw
  running build_ext
  building 'gssapi.raw.misc' extension
  creating build/temp.linux-x86_64-cpython-38
  creating build/temp.linux-x86_64-cpython-38/gssapi
  creating build/temp.linux-x86_64-cpython-38/gssapi/raw
  gcc -pthread -Wno-unused-result -Wsign-compare -DDYNAMIC_ANNOTATIONS_ENABLED=1 -DNDEBUG -O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -D_GNU_SOURCE -fPIC -fwrapv -O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -D_GNU_SOURCE -fPIC -fwrapv -O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -D_GNU_SOURCE -fPIC -fwrapv -fPIC -Igssapi/raw -I/usr/include/python3.8 -c gssapi/raw/misc.c -o build/temp.linux-x86_64-cpython-38/gssapi/raw/misc.o -DHAS_GSSAPI_EXT_H
  gssapi/raw/misc.c:43:10: fatal error: Python.h: No such file or directory
   #include "Python.h"
            ^~~~~~~~~~
  compilation terminated.
  error: command '/usr/bin/gcc' failed with exit code 1
  ----------------------------------------
  ERROR: Failed building wheel for gssapi
  Running setup.py clean for gssapi
Failed to build gssapi
ERROR: Could not build wheels for gssapi which use PEP 517 and cannot be installed directly
```

![Another one](/images/building_wheels_in_python/another_one.png)

Once again we ask our good-ol friend [google](https://stackoverflow.com/questions/21530577/fatal-error-python-h-no-such-file-or-directory)

What we installed before was just the Cpython interpreter. For building packages, however, we also need the Python development files(or simply the header files).

As you can see:

```bash
[root@b8e29bb9e124 /]# ls -l /usr/include/python3.8/
total 48
-rw-r--r-- 1 root root 47524 Aug 10 16:43 pyconfig-64.h
[root@b8e29bb9e124 /]# yum install python38-devel.x86_64
...

[root@b8e29bb9e124 /]# ls /usr/include/python3.8/
Python-ast.h       compile.h              genobject.h            node.h             pydtrace.h     pytime.h
Python.h           complexobject.h        graminit.h             object.h           pyerrors.h     rangeobject.h
_hashopenssl.h     context.h              grammar.h              objimpl.h          pyexpat.h      setobject.h
abstract.h         cpython                import.h               odictobject.h      pyfpe.h        sliceobject.h
asdl.h             datetime.h             internal               opcode.h           pyhash.h       structmember.h
ast.h              descrobject.h          interpreteridobject.h  osdefs.h           pylifecycle.h  structseq.h
bitset.h           dictobject.h           intrcheck.h            osmodule.h         pymacconfig.h  symtable.h
bltinmodule.h      dtoa.h                 iterobject.h           parsetok.h         pymacro.h      sysmodule.h
boolobject.h       dynamic_annotations.h  listobject.h           patchlevel.h       pymath.h       token.h
bytearrayobject.h  enumobject.h           longintrepr.h          picklebufobject.h  pymem.h        traceback.h
bytes_methods.h    errcode.h              longobject.h           py_curses.h        pyport.h       tracemalloc.h
bytesobject.h      eval.h                 marshal.h              pyarena.h          pystate.h      tupleobject.h
cellobject.h       fileobject.h           memoryobject.h         pycapsule.h        pystrcmp.h     typeslots.h
ceval.h            fileutils.h            methodobject.h         pyconfig-64.h      pystrhex.h     ucnhash.h
classobject.h      floatobject.h          modsupport.h           pyconfig.h         pystrtod.h     unicodeobject.h
code.h             frameobject.h          moduleobject.h         pyctype.h          pythonrun.h    warnings.h
codecs.h           funcobject.h           namespaceobject.h      pydebug.h          pythread.h     weakrefobject.h
[root@b8e29bb9e124 /]#
```

Hurray 😸 now we can see the "Python.h" file.

Once again let's install the package.(🤞)

```bash
[root@b8e29bb9e124 /]# /bin/python3.8 -m pip install gssapi
WARNING: Running pip install with root privileges is generally not a good idea. Try `python3.8 -m pip install --user` instead.
Collecting gssapi
  Using cached https://files.pythonhosted.org/packages/13/e7/dd88180cfcf243be62308707cc2f5dae4c726c68f30b9367931c794fda16/gssapi-1.8.3.tar.gz
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Installing backend dependencies ... done
    Preparing wheel metadata ... done
Collecting decorator
  Using cached https://files.pythonhosted.org/packages/d5/50/83c593b07763e1161326b3b8c6686f0f4b0f24d5526546bee538c89837d6/decorator-5.1.1-py3-none-any.whl
Building wheels for collected packages: gssapi
  Building wheel for gssapi (PEP 517) ... done
  Created wheel for gssapi: filename=gssapi-1.8.3-cp38-cp38-linux_x86_64.whl size=4712008 sha256=82b8dc449b2ed75e69eebf270a5bfc59e3ee9c304e1b715fcedcb070a6359b07
  Stored in directory: /root/.cache/pip/wheels/10/0d/d1/46523f6e9ac8f32799ea5aac89ab25114545ccc4cf92f0be7c
Successfully built gssapi
Installing collected packages: decorator, gssapi
Successfully installed decorator-5.1.1 gssapi-1.8.3
```

Finally, it's installed!!! 🐱🐱🐱

Let's check it.

```bash
[root@b8e29bb9e124 /]# /bin/python3.8
Python 3.8.17 (default, Aug 10 2023, 12:50:17)
[GCC 8.5.0 20210514 (Red Hat 8.5.0-20)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import gssapi
>>>
```

### Building wheel

To build a wheel (which is a glorified zip file) you need to use the `pip` command.

```bash
[root@b8e29bb9e124 /]# /bin/python3.8 -m pip wheel gssapi --no-cache
Collecting gssapi
  Downloading https://files.pythonhosted.org/packages/13/e7/dd88180cfcf243be62308707cc2f5dae4c726c68f30b9367931c794fda16/gssapi-1.8.3.tar.gz (94kB)
     |████████████████████████████████| 102kB 1.3MB/s
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Installing backend dependencies ... done
    Preparing wheel metadata ... done
Collecting decorator
  File was already downloaded /decorator-5.1.1-py3-none-any.whl
Skipping decorator, due to already being wheel.
Building wheels for collected packages: gssapi
  Building wheel for gssapi (PEP 517) ... done
  Created wheel for gssapi: filename=gssapi-1.8.3-cp38-cp38-linux_x86_64.whl size=4712077 sha256=f59219bf3a05773aae8ec51769946d79714eafc0125492b3562bbf851cfff09f
  Stored in directory: /
Successfully built gssapi
[root@b8e29bb9e124 /]#
[root@b8e29bb9e124 /]#
[root@b8e29bb9e124 /]# ls /*.whl
/decorator-5.1.1-py3-none-any.whl  /gssapi-1.8.3-cp38-cp38-linux_x86_64.whl
[root@b8e29bb9e124 /]#
```

And friends that's how you build a wheel.

### _NOTE_

In some cases, if Python as well as the Python header files are installed in a different location than /usr/include/ then GCC won't be able to find it we need to explicitly tell GCC where to look using CFLAGS.

```bash
export CFLAGS=-I<path_to_python_header>\
# for example
export CFLAGS=-I/usr/include/python3.8
```

### The last thing!!

Now that I built the wheel I needed to get the wheel from the container and give it to my friend. This can be easily done by mounting volume in podman using the _-v_ flag.

Read this post for more info: [working-container-storage-library-and-tools-red-hat-enterprise-linux](https://www.redhat.com/en/blog/working-container-storage-library-and-tools-red-hat-enterprise-linux).

### Conclusion

So we have seen the nits and pitfalls of building wheels in Python. This also explains how Wheels solved one of the biggest problems of the Python echo system which is package distribution(yes yes I know that package dependency is a whole other hell which is for another article in itself).
