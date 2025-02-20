## Benchmarking

This repository contains some tooling to help automate benchmarking, particularly
for the regalloc model (as of right now). The current benchmarking tooling works
with the llvm test suite, and the chromium performance tests.

## Initial Setup

Make sure you have local checkouts of the repositories that are used:
```bash
cd ~/
git clone https://github.com/llvm/llvm-project
git clone https://github.com/google/ml-compiler-opt
```
And for benchmarking using the llvm-test-suite:
```bash
git clone https://github.com/llvm-test-suite
```

For acquiring the chromium source code, please see [their](https://chromium.googlesource.com/chromium/src/+/main/docs/linux/build_instructions.md)
documentation and follow it up by running hooks. You don't need to setup any
builds as the `benchmark_chromium.py` script does that automatically.

Make sure that you have a local copy of libtensorflow:
```bash
mkdir ~/tensorflow
wget --quiet https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-cpu-linux-x86_64-1.15.0.tar.gz
tar xfz libtensorflow-cpu-linux-x86_64-1.15.0.tar.gz -C ~/tensorflow
```

And make sure you have installed all of the necessary python packages:
```bash
pipenv sync --categories "packages ci" --system
```

## Benchmarking the LLVM test suite

You can use the `benchmark_llvm_test_suite.py` python script in order to
automatically configure everything to run a benchmark using the latest released
regalloc model:
```bash
cd ~/ml-compiler-opt
PYTHONPATH=$PYTHONPATH:. python3 ./compiler_opt/benchmark/benchmark_llvm_test_suite.py \
  --advisor=release \
  --compile_llvm \
  --compile_testsuite \
  --llvm_build_path=~/llvm-build \
  --llvm_source_path=~/llvm-project/llvm \
  --llvm_test_suite_path=~/llvm-test-suite \
  --llvm_test_suite_build_path=~/llvm-test-suite/build \
  --nollvm_use_incremental \
  --model_path="download" \
  --output_path=./results.json \
  --perf_counter=INSTRUCTIONS \
  --perf_counter=MEM_UOPS_RETIRED:ALL_LOADS \
  --perf_counter=MEM_UOPS_RETIRED:ALL_STORES \
  --tensorflow_c_lib_path=~/tensorflow
```

This will output a bunch of test information to `./results.json` that can then
be used later on for downstream processing and data analysis.

An explanation of the flags:
* `--advisor` - This flag specifies the register allocation eviction advisor that
is used by LLVM when compiling the test suite. It can be set to either `release`
or `default` depending upon if you want to test the model specified in the
`--model_path` flag or if you want to test the default register allocation eviction
behavior to grab a baseline measurement.
* `--compile_llvm` - This is a boolean flag (can also be set to `--nocompile_llvm`)
that specifies whether or not to compile LLVM.
* `--compile_testsuite` - Specifies whether or not to compile the test suite.
* `--llvm_build_path` - The path to place the LLVM build in that will be used.
This directory will be deleted and remade if the `--nollvm_use_incremental` flag
is set.
* `--llvm_source_path` - The path to the LLVM source. This cannot be the root path
to the LLVM monorepo, it specifically needs to be the path to the llvm
subdirectory within that repository.
* `--llvm_test_suite_path` - The path to the llvm-test-suite
* `--llvm_test_suite_build_path` - The path to place the build for the
llvm-test-suite. Similar behavior to the LLVM build path.
* `llvm_use_incremental` - Whether or not to do an incremental build of LLVM.
If you alread have all the correct compilation flags setup for running MLGO
with LLVM, you can set this flag and you should get an extremely fast LLVM
build as the only thing changing is the release mode regalloc model.
* `model_path` - The path to the regalloc model. If this is set to "download",
it will automatically grab the latest model from the ml-compiler-opt Github.
If it is set to "" or "autogenerate", it will use the autogenerated model.
* `output_path` - The path to the output file (in JSON format)
* `perf_counter` - A flag that can be specified multiple times that takes in
performance counters in the libpfm format. There can only be up to three
performance counters specified due to underlying limitations in Google
benchmark.
* `tensorflow_c_lib_path` - The path to the tensorflow c library if you aren't
doing an incremental LLVM build.
* `tests_to_run` - This specifies the LLVM microbenchmarks to run relative to
the microbenchmarks library in the LLVM test suite build directory. The default
values for this flag should be pretty safe and produce good results.

You can also get detailed information on each flag by only passing the `--help`
flag to the script. You can also see the default values here as a lot of the
flags set in the example above are just to their default value.

## Benchmarking Chromium

You can use the `benchmark_chromium.py` script in order to run chromium
benchmarks based on test description JSON files.

Example:
```bash
cd ~/ml-compiler-opt
PYTHONPATH=$PYTHONPATH:. python3 ./compiler_opt/benchmark/benchmark_chromium.py \
  --advisor=release \
  --chromium_build_path=./out/Release \
  --chromium_src_path=~/chromim/src \
  --compile_llvm \
  --compile_tests \
  --depot_tools_path=~/depot_tools \
  --llvm_build_path=~/llvm-build \
  --llvm_source_path=~/llvm-project/llvm \
  --nollvm_use_incremental \
  --model_path="download" \
  --num_threads=32 \
  --output_file=./output-chromium-testing.json \
  --perf_counters=mem_uops_retired.all_loads \
  --perf_counters=mem_uops_retired.all_stores \
  --tensorflow_c_lib_path=~/tensorflow
```

Several of the flags here are extremely similar to/the same as the flags
for the llvm test suite, so only the flags unique to the chromium script
will be highlighted here.
* `--chromium_build_path` - The path to place the chromium build in. This path
is relative to the chromium source path.
* `-chromium_src_path` - The path to the root of the chromium repository (ie
`./src/` where you ran `fetch --nohooks`)
* `--depot_tools_path` - The path yo your depot tools checkout.
* `--num_threads` - enables parallelism for running the tests. Make sure to use
this with caution as it can add a lot of noise to your benchmarks depending
upon what specifically you are doing.
* `--perf_counters` - similar to the llvm test suite perf counters, but instead
of being in the libpfm format, they're perf counters as listed in `perf list`.
* `--test_description` - Can be declared multiple times if you have custom test
descriptions that you want to run, but the default works well, covers a broad
portion of the codebase, and has been specifically designed to minimize run
to run variability.

### Generating Chromium Test Descriptions

To generate custom test descriptions for gtest executables (ie the test
executables that are used by chromium), you can use the `list_gtests.py` script.
This script doesn't need to be used for running the chromium performance tests
unless you are interested in adjusting the currently set test descriptions
available in `/compiler_opt/benchmark/chromium_test_descriptions` or are
interested in using tests from a different project that also uses gtest.

Example:
```bash
PYTHONPATH=$PYTHONPATH:. python3 ./compiler_opt/benchmark/list_gtests.py \
  --gtest_executable=/path/to/executable \
  --output_file=test.json \
  --output_type=json
```

Flags:
* `--gtest_executable` - The path to the gtest executable from which to extract
a list of tests
* `--output_file` - The path to the file to output all of the extracted test names
too
* `--output_type` - Either JSON or default. JSON packages everything nicely into
a JSON format, and default just dumps the test names separated by line breaks.

There is also a utility `filter_tests.py` that allows for filtering the
individual tests available in a test executable, making sure that they exist
(sometimes tests that pop up when listing all the gtests don't run when passed
through `-gtest_filter`) and that they don't fail (some tests require setups
including GUI/GPUs).

Example:
```bash
PYTHONPATH=$PYTHONPATH:. python3 ./compiler_opt/benchmark/filter_tests.py \
  --input_tests=./compiler_pt/benchmark/chromium_test_descriptions/browser_tests.json \
  --output_tests=./browser_tests_filtered.json \
  --num_threads=32 \
  --executable_path=/chromium/src/out/Release/browser_tests
```

Flags:
* `--input_tests` - The path to the test description generated by
`list_gtests.py` to be filtered.
* `--output_tests` - The path to where the new filtered output test suite
description should be placed.
* `--num_threads` - The number of threads to use when running tests to see if
they exist/whether or not they pass.
* `--executable_path` - The path to the gtest executable that the test suite
description corresponds to.

TODO(boomanaiden154): investigate why some of the tests listed by the
executable later can't be found when using `--gtest_filter`.

## Comparing Benchmarks

To compare benchmark runs, you can use the `benchmark_report_converter.py` script.
Let's say you have two benchmark runs (they need to be done with the same set
of tests), `baseline.json` and `experimental.json` from the llvm test suite
benchmarking script with the performance counter `INSTRUCTIONS` enabled. You can get 
a summary comparison with the following command:
```bash
PYTHONPATH=$PYTHONPATH:. python3 ./compiler_opt/benchmark/benchmark_report_converter.py \
--base=baseline.json \
--exp=experimental.json \
--counters=INSTRUCTIONS \
--out=reports.csv
```
This will create `reports.csv` with a line for each test that contains information
about the differences in performance counters for that specific test.
