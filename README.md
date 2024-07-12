# Loadable Module Development Kit - Standalone Kit using only Stable APIs.

LMDK (Loadable Module Development Kit) is a standalone package that allows building a loadable module for the SOF. It operates independently from SOF FW, but it contains all necessary data structure definitions required for interaction with it.

Loadable modules constructed with LMDK provide functionality identical to that of IADK (Intel Audio Development Kit) modules. IADK modules are developed in C++, and their interface is grounded on classes with virtual methods. Given that SOF FW is written in C, an adapter is required for IADK modules to provide the required interfaces. Loadable modules developed using LMDK utilize a C language interface, which is the reason they are also referred to as native loadable modules.


## Preparing a loadable module

All prepared module files should be situated in the `modules/<module_name>` directory. The most basic loadable module must encompass the following components:


#### API version definition
A declaration of the API version used is required. This is accomplished using the `DECLARE_LOADABLE_MODULE_API_VERSION` macro, which accepts the module name as a parameter.

```C
DECLARE_LOADABLE_MODULE_API_VERSION(example);
```
It places at the beginning of the .text section a structure identifying the version of the API used by the module. During the module loading process, the SOF verifies if the declared API version is supported. The API version is declared in the header files and should not be altered independently.


#### Module interface

The `struct module_interface` structure represents the module interface. It contains pointers to functions provided by the module.

```C
static struct module_interface example_module_interface = {
	.init = example_init,
	.process = example_process,
	.set_processing_mode = example_set_processing_mode,
 	.get_processing_mode = example_get_processing_mode,
	.set_configuration = example_set_configuration,
	.get_configuration = example_get_configuration,
	.reset = example_reset,
};
```

Currently, the module can provide one of three sample processing function. The `process` function is recommended and also allows processing in DP (Data Processing) mode. The functions `process_audio_stream` and `process_raw_data` are deprecated and will be phased out in the future. A comprehensive description of the function with parameters is situated in the `module/interface.h` header file. All functions provided by the module accepts as parameters a pointer to the `struct processing_module` structure, which represents the module. A module can store a pointer to its private data using the `module_get_private_data` function and retrieve it using the `module_set_private_data` function.


#### Module entry point

```C
static const struct native_system_agent *system_agent;

static void* example_entry_point(const struct byte_array* mod_cfg,
				 void* reserved,
				 const struct native_system_agent* const* agent)
{
	system_agent = *agent;

	return &example_module_interface;
}
```

This function serves as the entry point for the module. It is invoked when each instance of the module is created. The `mod_cfg` parameter carries the module configuration passed via IPC. The `agent` parameter conveys a pointer to the system agent. The function yields a pointer to the module interface. Subsequently, the `init` function from the returned module interface is invoked.

#### Module manifest definition

```C
__attribute__((section(".module")))
const struct sof_man_module_manifest example_module_manifest = {
    .module = {
        .name = "EXAMPLE",
        .uuid = {0x82, 0x68, 0x5a, 0x91, 0x0b, 0xda, 0x4b, 0x23,
		 0xa6, 0x2a, 0x4c, 0x2e, 0x77, 0xf7, 0xa9, 0x14},
        .entry_point = (uint32_t)example_entry_point,
        .type = {
            .load_type = SOF_MAN_MOD_TYPE_MODULE,
            .domain_ll = 1
        },
        .affinity_mask = 3,
    }
};
```

The structure of a module manifest specifies its name, GUID, entry point, and fundamental configuration. This structure is utilized by the `rimage` during the library preparation process. The `rimage` populates the remaining fields of this structure (offsets in the output file, hash), and in conjunction with the configuration from the toml file, incorporates them into the final image.


#### Preparing the module makefile

LMDK utilizes the CMake build tool. Consequently, the preparation of a CMakeLists.txt file for the module is necessary.

```cmake
target_sources(example PRIVATE example.c)

set_target_properties(example PROPERTIES
    HPSRAM_ADDR "0xa06a1000"
)
```

The `HPSRAM_ADDR` property designates the base address where the module will be linked during the compilation process. The module will be situated at this address in the DSP memory during its loading phase. It is essential to specify a valid address that resides within the DSP address space. The address spaces of individual modules should not intersect. This is verified by rimage during the preparation of the final image.


#### Preparing a loadable library

The final step before initiating the compilation process is to prepare the `CMakeLists.txt` file for the library in the `libraries/<lib_name>` directory. This file dictates which modules will be incorporated into the resulting image. A library can comprise one or more modules. It is essential to specify the path to the `toml` file that contains the configuration of the prepared modules. The format of this file is identical to that utilized by SOF.

```cmake
cmake_minimum_required(VERSION 3.20)
set(CMAKE_TOOLCHAIN_FILE "${CMAKE_CURRENT_LIST_DIR}/../../cmake/xtensa-toolchain.cmake")

project(example_lib)

# list of modules to be built and included into this loadable library
set(MODULES_LIST example)

# toml file for rimage to generate manifets
set(TOML "${CMAKE_CURRENT_LIST_DIR}/example.toml")

include(../../cmake/build.cmake)
```


## Building of loadable library

The designers of LMDK have prepared a Python script to build the loadable library. To build an example loadable library, execute the following:

```bash
$ python scripts/lmdk/libraries_build.py -l example_lib -k "/path/to/signing/key.pem"
```

The `-l` parameter designates the name of the library to be built. The `-k` parameter specifies the path to a file containing a private key, which will be utilized to sign the resulting image.
