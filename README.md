# AP-GLLVM

<!-- add hyperlinks here  -->
**Description**
A modified version of gllvm which works on the ArduPilot build system 

See ```gllvm.md``` for the original gllvm README

**Logistical Differences**
- Note that this version of gllvm must be installed with 
    ```bash
    go install github.com/anonymous-submission-25/ap-gllvm/cmd/...@v1.0.4
    ```

- We have also updated the minimum verison of go to 1.17 from 1.16 since version 1.16 is no longer supported

**Modification**
- The only executables that are affected by these modifications are ```gclang``` and ```gclang++```

- All modifications to the source are in the file ```shared/parser.go``` in the function getArtifactNames.
    - The key issue is that gllvm aims to compile object files and bitcode files separately, before adding the path of the bitcode file to the object file which will be linked in the final executable. However, the gllvm system assumes 2 invariants to hold during the bitcode path addition step, which do not hold for the ArduPilot build system (*for more on the ArduPilot build system, see documentation for [waf](https://waf.io/book/)*): <!-- add a hyperlink here for waf --> 
        1. All object files must placed in the same directory that they are compiled in
        2. For every ```.c``` or ```.cpp``` to be compiled, in the format ```filename.cpp```, its corresponding object file should be called ```filename.o``` (these files exist mostly in the subdirectories of `ardupilot/libraries` or `ardupilot/[vehicle]`)

    - The first invariant does not hold because the waf executable uses ```ardupilot/build/sitl``` *(or whichever board is being used)* as its working directory for the entire build process rather than ```cd```ing into the directory where object files will be stored before compiling them *(ex. ```ardupilot/build/sitl/libraries/AC_CustomControl/```)*. The second invariant does not hold because most or all ArduPilot object files are compiled into a format like filename.someextnesion.o, where we are compiling filename.cpp. For example, ```AC_CustomControl.cpp``` compiled to ```AC_CustomControl.cpp.0.o``` rather than ```AC_CustomControl.o```.

    - A secondary issue is that there were a few instances where a different version of ```filename.cpp``` existed in multiple different directories. However, since the entire compilation process was taking place in ```ardupilot/build/sitl```, the ```filename.bc``` files were being places in the same directory, causing one or more to be overwritten and lost.
    
    - Our solution involves modifying the getArtifactNames which returns the name of the object file that has just been generated and the corresponding bitcode file that gllvm will generate in the next step in order to attach it to the object file. We modify the function to keep track of the exact file name of the object file by referring the the output argument of the compile command, rather than the input argument as was the original functionality. Further, we obtain the relative path of the compile command argument and add this to both of our return variables (object file name and bitcode file name). We add to the object file to ensure that gllvm can find the object file to link the corresponding bitcode file. We add it to the bitcode file name to ensure that it is placed in the same directory as the object file, thus negating the problem of same name bitcode files overwritting one another.
 
    - For example:
        - If we are compiling the file `ardupilot/libraries/AP_HAL/Storage.cpp`, it will be compiled to the object file: `ardupilot/build/sitl/libraries/AP_HAL/Storage.cpp.0.o`
        - However, the original implementation of GLLVM would compile the corresponding bitcode file as `ardupilot/build/sitl/.Storage.cpp.o.bc` rather than `ardupilot/build/sitl/libraries/AP_HAL/.Storage.cpp.o.bc`
        - It would proceed to search for the object file `ardupilot/build/sitl/Storage.o` to attach the bitcode file to
            - Unfortunately the file `ardupilot/build/sitl/Storage.o` does not exist
            - Additionally, there exists a file `ardupilot/libraries/AP_HAL_EMPTY/Storage.cpp` which has the same name but exists in a different directory
            - so when we compile it to its corresponding bitcode file, the version of `ardupilot/build/sitl/.Storage.cpp.o.bc` corresponding to the `AP_HAL` directory is overwritten, in favor of the same file name as corresponding to the `AP_HAL_EMPTY` directory
        - In our solution, we compile the bitcode file in `ardupilot/build/sitl/libraries/AP_HAL/.Storage.cpp.o.bc` rather than `ardupilot/build/sitl/.Storage.cpp.o.bc` (implemented in `ap-gllvm/shared/parser.go:516-529, 570`  using variable `rel_path`)
        - And we attach the bitcode file to the object file `ardupilot/build/sitl/libraries/AP_HAL/Storage.cpp.0.o` rather than `ardupilot/build/sitl/Storage.o` (implemented in `ap-gllvm/shared/parser.go:516-537, 571-575` using variables `rel_path` and `of_ext`)

