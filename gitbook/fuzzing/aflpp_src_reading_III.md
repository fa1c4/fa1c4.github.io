# AFL++ Source Code Reading III - Instrumentation

本文解析 AFL++ 插桩部分的源代码

AFL++ 从 3.00c 版本开始统一所有插桩编译器到 `afl-cc`, 虽然编译 AFL++ 之后依然有 `afl-clang*` 之类的编译器, 但都是链接到 `afl-cc` 的链接文件, 实际调用的都是 `afl-cc`. AFL++ 支持的插桩方式很多, 本篇主要关注 llvm mode. 



## Overview

AFL++ 支持的插桩选择如下

| Instrumentation         | Desription                                                   |
| ----------------------- | ------------------------------------------------------------ |
| CmpLog                  | Enables logging of comparison operands in a shared memory. Used by various mutators, like redqueen [2]. |
| GCC-based               | `afl-gcc-fast` and `afl-g++-fast`. True compiler-level instrumentation. |
| Injection               | Proof-of-concept implementation to additionally hunt for injection vulnerabilities. It works by instrumenting calls to specific functions and parsing the query parameter for a specific unescaped dictionary string, and if detected, crashes the target. |
| Partial Instrumentation | Testing complex programs where only a part of the program is the fuzzing target, it often helps to only instrument the necessary parts of the program, leaving the rest uninstrumented. |
| laf-intel               | Make some code transformations that help AFL++ to enter conditional blocks, where conditions consist of comparisons of large values. |
| Fast LLVM-based         | `afl-clang-fast` and `afl-clang-fast++`. True compiler-level instrumentation. Requires llvm-3.8 up to 17. |
| lto                     | Collision free instrumentation and is faster than PCGUARD. Requires LLVM 12+. |
| persistent mode         | AFL++ fuzzes a target multiple times in a single forked process, instead of forking a new process for each fuzz execution. Persistent mode requires that the target can be called in one or more functions, and that its state can be completely reset so that multiple calls can be performed without resource leaks, and that earlier runs will have no impact on future runs. |



llvm mode 的核心源文件

+ **afl-llvm-common.***: LLVM 插桩的公共辅助代码, 封装多个插桩 Pass 共用的函数和数据结构, 比如插桩点管理、辅助工具函数等. 
+ **afl-llvm-pass.so.cc**: LLVM 插桩 Pass 的实现文件, 定义在 LLVM 编译链中插入覆盖率代码的逻辑, 主要负责插入计数点, 记录执行路径信息. 
+ **afl-compiler-rt.o.c (afl-llvm-rt.o.c in old afl)**: AFL++ 的运行时库文件, 提供计数器更新、共享内存操作等插桩时所需的底层功能



## afl-compiler-rt.o.c

### 全局变量

```c
// buffer for share memory data & execution counter 
static u8  __afl_area_initial[MAP_INITIAL_SIZE];

// __afl_area_ptr* points to memory where instrumentation code writing the data
static u8 *__afl_area_ptr_dummy = __afl_area_initial;
static u8 *__afl_area_ptr_backup = __afl_area_initial;
u8        *__afl_area_ptr = __afl_area_initial;

// point to mutation dictionary
u8        *__afl_dictionary;
u32 __afl_dictionary_len;

// points to testcase buffer 
u8        *__afl_fuzz_ptr;
// the length of testcase input buffer
static u32 __afl_fuzz_len_dummy;
u32       *__afl_fuzz_len = &__afl_fuzz_len_dummy;

// flag shows whether enables sharedmem fuzzing
int        __afl_sharedmem_fuzzing __attribute__((weak));

// the last location of instrumentation
u32 __afl_final_loc;
// the first location of valid instrumentation
u32 __afl_first_final_loc;

// share memory size default 2^16 == 64 KB == 65536
u32 __afl_map_size = MAP_SIZE;
// points to mappings of share memory coverage
u64 __afl_map_addr;

// compatible with old version of forkserver (ignored)
u32 __afl_old_forkserver;

//...
    
/* 1 if we are running in afl, and the forkserver was started, else 0 */
u32 __afl_connected = 0;

// for the __AFL_COVERAGE_ON/__AFL_COVERAGE_OFF features to work:
int        __afl_selective_coverage __attribute__((weak)); // whether enables selective coverage
int        __afl_selective_coverage_start_off __attribute__((weak)); // disables selective coverage from beginning
static int __afl_selective_coverage_temp = 1; // temp state of selective coverage

#if defined(__ANDROID__) || defined(__HAIKU__) || defined(NO_TLS)
PREV_LOC_T __afl_prev_loc[NGRAM_SIZE_MAX];
PREV_LOC_T __afl_prev_caller[CTX_MAX_K];
u32        __afl_prev_ctx;
#else
__thread PREV_LOC_T __afl_prev_loc[NGRAM_SIZE_MAX]; // context aware coverage
__thread PREV_LOC_T __afl_prev_caller[CTX_MAX_K]; // call stack aware coverage 
__thread u32        __afl_prev_ctx; // ID of context
#endif

// mapping for following current comparing instruction
struct cmp_map *__afl_cmp_map;
struct cmp_map *__afl_cmp_map_backup;
// log length of comparing instructions
static u8 __afl_cmplog_max_len = 32;  // 16-32

// child process ID for forkserver mode
static s32 child_pid;
// pointer for recovering signal
static void (*old_sigterm_handler)(int) = 0;

/* Running in persistent mode? */
static u8 is_persistent;

/* Are we in sancov mode? */
static u8 _is_sancov;

/* Debug? */
/*static*/ u32 __afl_debug;

/* Already initialized markers to avoid re-initialization */
u32 __afl_already_initialized_shm;
u32 __afl_already_initialized_forkserver;
u32 __afl_already_initialized_first;
u32 __afl_already_initialized_second;
u32 __afl_already_initialized_early;
u32 __afl_already_initialized_init;

/* Dummy pipe for area_is_valid() */
static int __afl_dummy_fd[2] = {2, 2};
```



### 结构体

```c
// information for PCGuard coverage
struct afl_module_info_t {

  // A unique id starting with 0
  u32 id;

  // Name and base address of the module
  char     *name;
  uintptr_t base_address;

  // PC Guard start/stop
  u32 *start;
  u32 *stop;

  // PC Table begin/end
  const uintptr_t *pcs_beg;
  const uintptr_t *pcs_end;

  u8 mapped; // whether mapped or not

  afl_module_info_t *next; // next pointer: link data structure organizes multi modules

};

// assists for coverage
typedef struct {

  uintptr_t PC, PCFlags;

} PCTableEntry;

// header pointer points to module coverage link
afl_module_info_t *__afl_module_info = NULL;

// pc mapping table size
u32        __afl_pcmap_size = 0;
// points to program counter array
uintptr_t *__afl_pcmap_ptr = NULL;

// filtering the area where we want to skip instrumentation
typedef struct {

  uintptr_t start; // start of skip area
  u32       len; // length of skip area

} FilterPCEntry;
u32            __afl_filter_pcs_size = 0; // number of areas
FilterPCEntry *__afl_filter_pcs = NULL; // filtering arrays pointer
u8            *__afl_filter_pcs_module = NULL; // filtering module pointer
```



### Share Memory

```c
/* SHM fuzzing setup for store testcase input */
static void __afl_map_shm_fuzz() {
  // get the id of share memory of input 
  char *id_str = getenv(SHM_FUZZ_ENV_VAR);
  // ...
  if (id_str) {

    u8 *map = NULL;
      
// selet share memory mode by USEMMAP
#ifdef USEMMAP
    const char *shm_file_path = id_str;
    int         shm_fd = -1;

    /* create the shared memory segment as if it was a file */
    shm_fd = shm_open(shm_file_path, O_RDWR, DEFAULT_PERMISSION);
    if (shm_fd == -1) {

      fprintf(stderr, "shm_open() failed for fuzz\n");
      send_forkserver_error(FS_ERROR_SHM_OPEN);
      exit(1);

    }

    map =
        (u8 *)mmap(0, MAX_FILE + sizeof(u32), PROT_READ, MAP_SHARED, shm_fd, 0);
#else
    u32 shm_id = atoi(id_str);
    map = (u8 *)shmat(shm_id, NULL, 0);
#endif
    // ...
      
    __afl_fuzz_len = (u32 *)map;
    __afl_fuzz_ptr = map + sizeof(u32);

    // ...

  } else {

    fprintf(stderr, "Error: variable for fuzzing shared memory is not set\n");
    send_forkserver_error(FS_ERROR_SHM_OPEN);
    exit(1);

  }

}


/* SHM setup. */
static void __afl_map_shm(void) {

  if (__afl_already_initialized_shm) return;
  __afl_already_initialized_shm = 1;

  // if we are not running in afl ensure the map exists
  if (!__afl_area_ptr) { __afl_area_ptr = __afl_area_ptr_dummy; }

  char *id_str = getenv(SHM_ENV_VAR);

  if (__afl_final_loc) {

    // adjust map_size by final_loc
    __afl_map_size = __afl_final_loc + 1;  // as we count starting 0

    // ...
	// remind to set larger MAP_SIZE
    if (__afl_final_loc > MAP_SIZE) {

      char *ptr;
      u32   val = 0;
      if ((ptr = getenv("AFL_MAP_SIZE")) != NULL) { val = atoi(ptr); }
      if (val < __afl_final_loc) {

        if (__afl_final_loc > MAP_INITIAL_SIZE && !getenv("AFL_QUIET")) {

          fprintf(stderr,
                  "Warning: AFL++ tools might need to set AFL_MAP_SIZE to %u "
                  "to be able to run this instrumented program if this "
                  "crashes!\n",
                  __afl_final_loc);

        }

      }

    }

  }

  // check if disable sharedmem fuzzing
  if (__afl_sharedmem_fuzzing && (!id_str || !getenv(SHM_FUZZ_ENV_VAR) ||
                                  fcntl(FORKSRV_FD, F_GETFD) == -1 ||
                                  fcntl(FORKSRV_FD + 1, F_GETFD) == -1)) {

	// ...
    __afl_sharedmem_fuzzing = 0;

  }

  // init map pointers 
  if (!id_str) {

    u32 val = 0;
    u8 *ptr;

    if ((ptr = getenv("AFL_MAP_SIZE")) != NULL) { val = atoi(ptr); }

    if (val > MAP_INITIAL_SIZE && val > __afl_final_loc) {

      __afl_map_size = val;

    } else {

      if (__afl_first_final_loc > MAP_INITIAL_SIZE) {

        // done in second stage constructor
        __afl_map_size = __afl_first_final_loc;

      } else {

        __afl_map_size = MAP_INITIAL_SIZE;

      }

    }

    if (__afl_map_size > MAP_INITIAL_SIZE && __afl_final_loc < __afl_map_size) {

      __afl_final_loc = __afl_map_size;

    }

	// ...

  }

  /* If we're running under AFL, attach to the appropriate region, replacing the
     early-stage __afl_area_initial region that is needed to allow some really
     hacky .init code to work correctly in projects such as OpenSSL. */
  // ...

  // share memory mapping to fuzzer mmap
  if (id_str) {

    if (__afl_area_ptr && __afl_area_ptr != __afl_area_initial &&
        __afl_area_ptr != __afl_area_ptr_dummy) {

      if (__afl_map_addr) {

        munmap((void *)__afl_map_addr, __afl_final_loc);

      } else {

        free(__afl_area_ptr);

      }

      __afl_area_ptr = __afl_area_ptr_dummy;

    }

#ifdef USEMMAP
    const char    *shm_file_path = id_str;
    int            shm_fd = -1;
    unsigned char *shm_base = NULL;

    /* create the shared memory segment as if it was a file */
    shm_fd = shm_open(shm_file_path, O_RDWR, DEFAULT_PERMISSION);
    if (shm_fd == -1) {

      fprintf(stderr, "shm_open() failed\n");
      send_forkserver_error(FS_ERROR_SHM_OPEN);
      exit(1);

    }

    /* map the shared memory segment to the address space of the process */
    if (__afl_map_addr) {

      shm_base =
          mmap((void *)__afl_map_addr, __afl_map_size, PROT_READ | PROT_WRITE,
               MAP_FIXED_NOREPLACE | MAP_SHARED, shm_fd, 0);

    } else {

      shm_base = mmap(0, __afl_map_size, PROT_READ | PROT_WRITE, MAP_SHARED,
                      shm_fd, 0);

    }

    close(shm_fd);
    shm_fd = -1;

    if (shm_base == MAP_FAILED) {
	  // ...
    }

    __afl_area_ptr = shm_base;
#else
    u32 shm_id = atoi(id_str);

    if (__afl_map_size && __afl_map_size > MAP_SIZE) {

      u8 *map_env = (u8 *)getenv("AFL_MAP_SIZE");
      if (!map_env || atoi((char *)map_env) < MAP_SIZE) {

        fprintf(stderr, "FS_ERROR_MAP_SIZE\n");
        send_forkserver_error(FS_ERROR_MAP_SIZE);
        _exit(1);

      }

    }

    __afl_area_ptr = (u8 *)shmat(shm_id, (void *)__afl_map_addr, 0);

    /* Whooooops. */
	// ...

#endif

    /* Write something into the bitmap so that even with low AFL_INST_RATIO,
       our parent doesn't give up on us. */
	// write 1 into __afl_area_ptr[0] (shm[0]) to avoid being regarded as no coverage
    __afl_area_ptr[0] = 1;

  } else if ((!__afl_area_ptr || __afl_area_ptr == __afl_area_initial) &&

             __afl_map_addr) {

    __afl_area_ptr = (u8 *)mmap(
        (void *)__afl_map_addr, __afl_map_size, PROT_READ | PROT_WRITE,
        MAP_FIXED_NOREPLACE | MAP_SHARED | MAP_ANONYMOUS, -1, 0);

    if (__afl_area_ptr == MAP_FAILED) {

      fprintf(stderr, "can not acquire mmap for address %p\n",
              (void *)__afl_map_addr);
      send_forkserver_error(FS_ERROR_SHM_OPEN);
      exit(1);

    }
    // if instrumentation needs more space to store then malloc larger area
  } else if (__afl_final_loc > MAP_INITIAL_SIZE &&

             __afl_final_loc > __afl_first_final_loc) {

    if (__afl_area_initial != __afl_area_ptr_dummy) {

      free(__afl_area_ptr_dummy);

    }

    __afl_map_size = __afl_final_loc + 1;
    __afl_area_ptr_dummy = (u8 *)malloc(__afl_map_size);
    __afl_area_ptr = __afl_area_ptr_dummy;

    if (!__afl_area_ptr_dummy) {

      fprintf(stderr,
              "Error: AFL++ could not acquire %u bytes of memory, exiting!\n",
              __afl_final_loc);
      exit(-1);

    }

  }  // else: nothing to be done

  __afl_area_ptr_backup = __afl_area_ptr;
  // ...

  if (__afl_selective_coverage) {

    if (__afl_map_size > MAP_INITIAL_SIZE) {

      __afl_area_ptr_dummy = (u8 *)malloc(__afl_map_size);

    }

    if (__afl_area_ptr_dummy) {

      if (__afl_selective_coverage_start_off) {

        __afl_area_ptr = __afl_area_ptr_dummy;

      }

    } else {

      fprintf(stderr, "Error: __afl_selective_coverage failed!\n");
      __afl_selective_coverage = 0;
      // continue;

    }

  }

  // share memory of comparing instruction log
  id_str = getenv(CMPLOG_SHM_ENV_VAR);
  // ...

  if (id_str) {

    // /dev/null doesn't work so we use /dev/urandom
    if ((__afl_dummy_fd[1] = open("/dev/urandom", O_WRONLY)) < 0) {

      if (pipe(__afl_dummy_fd) < 0) { __afl_dummy_fd[1] = 1; }

    }

#ifdef USEMMAP
    const char     *shm_file_path = id_str;
    int             shm_fd = -1;
    struct cmp_map *shm_base = NULL;

    /* create the shared memory segment as if it was a file */
    shm_fd = shm_open(shm_file_path, O_RDWR, DEFAULT_PERMISSION);
    if (shm_fd == -1) {

      perror("shm_open() failed\n");
      send_forkserver_error(FS_ERROR_SHM_OPEN);
      exit(1);

    }

    /* map the shared memory segment to the address space of the process */
    shm_base = mmap(0, sizeof(struct cmp_map), PROT_READ | PROT_WRITE,
                    MAP_SHARED, shm_fd, 0);
    if (shm_base == MAP_FAILED) {

      close(shm_fd);
      shm_fd = -1;

      fprintf(stderr, "mmap() failed\n");
      send_forkserver_error(FS_ERROR_SHM_OPEN);
      exit(2);

    }

    __afl_cmp_map = shm_base;
#else
    u32 shm_id = atoi(id_str);

    __afl_cmp_map = (struct cmp_map *)shmat(shm_id, NULL, 0);
#endif

    __afl_cmp_map_backup = __afl_cmp_map;

    if (!__afl_cmp_map || __afl_cmp_map == (void *)-1) {

      perror("shmat for cmplog");
      send_forkserver_error(FS_ERROR_SHM_OPEN);
      _exit(1);

    }

  }

#ifdef __AFL_CODE_COVERAGE
  // share memory of PCMAP (e.g. from PC to source code)
  char *pcmap_id_str = getenv("__AFL_PCMAP_SHM_ID");

  if (pcmap_id_str) {

    __afl_pcmap_size = __afl_map_size * sizeof(void *);
    u32 shm_id = atoi(pcmap_id_str);

    __afl_pcmap_ptr = (uintptr_t *)shmat(shm_id, NULL, 0);

    if (__afl_debug) {

      fprintf(stderr, "DEBUG: Received %p via shmat for pcmap\n",
              __afl_pcmap_ptr);

    }

  }

#endif  // __AFL_CODE_COVERAGE

  if (!__afl_cmp_map && getenv("AFL_CMPLOG_DEBUG")) {

    __afl_cmp_map_backup = __afl_cmp_map = malloc(sizeof(struct cmp_map));

  }

  // adjust CMPLOG length
  if (getenv("AFL_CMPLOG_MAX_LEN")) {

    int tmp = atoi(getenv("AFL_CMPLOG_MAX_LEN"));
    if (tmp >= 16 && tmp <= 32) { __afl_cmplog_max_len = tmp; }

  }

}
```





## afl-llvm-pass.so.cc

### AFLCoverage

```c
namespace {

#if LLVM_VERSION_MAJOR >= 11                        /* use new pass manager */
class AFLCoverage : public PassInfoMixin<AFLCoverage> { // <!> core pass class for instrumentation

 public:
  AFLCoverage() {

#else
class AFLCoverage : public ModulePass {

 public:
  static char ID;
  AFLCoverage() : ModulePass(ID) {

#endif

    initInstrumentList(); // init instrumentation allow/skip list

  }

#if LLVM_VERSION_MAJOR >= 11                        /* use new pass manager */
  PreservedAnalyses run(Module &M, ModuleAnalysisManager &MAM); // entry function
#else
  bool runOnModule(Module &M) override; // old entry function
#endif

 protected:
  uint32_t    ngram_size = 0; // ngram related
  uint32_t    ctx_k = 0; // context related 
  uint32_t    map_size = MAP_SIZE; // map_size
  uint32_t    function_minimum_size = 1; // function minimum size
  const char *ctx_str = NULL, *caller_str = NULL, *skip_nozero = NULL; // context related
  const char *use_threadsafe_counters = nullptr; // set threadsafe counters

};

}  // namespace
```



### Pass Register

```c
#if LLVM_VERSION_MAJOR >= 11                        /* use new pass manager */
extern "C" LLVM_ATTRIBUTE_WEAK PassPluginLibraryInfo llvmGetPassPluginInfo() {
  // use PassPluginLibraryInfo to register pass
  return {LLVM_PLUGIN_API_VERSION, "AFLCoverage", "v0.1",
          /* lambda to insert our pass into the pass pipeline. */
          [](PassBuilder &PB) {

  #if 1
    #if LLVM_VERSION_MAJOR <= 13
            using OptimizationLevel = typename PassBuilder::OptimizationLevel; // set optimization level
    #endif
    #if LLVM_VERSION_MAJOR >= 16
      #if LLVM_VERSION_MAJOR >= 20
            PB.registerPipelineStartEPCallback( // version 20+ register
      #else
            PB.registerOptimizerEarlyEPCallback( // version 16-19 register
      #endif
    #else
            PB.registerOptimizerLastEPCallback( // version 15- register
    #endif
                [](ModulePassManager &MPM, OptimizationLevel OL) {

                  MPM.addPass(AFLCoverage()); // export pass AFLCoverage

                });

  /* TODO LTO registration */
  #else
	// ...

  #endif

          }}; // return \{\{*}};

} // llvmGetPassPluginInfo

#else

char AFLCoverage::ID = 0;
#endif
```



### Instrumentation "Main Function"

`AFLCoverage::run()` or `AFLCoverage::runOnModule` 是插桩 pass 的入口函数用于遍历 `Module &M` 并插入检查点. 其余逻辑见下面代码中的注释

```c
#if LLVM_VERSION_MAJOR >= 11                        /* use new pass manager */
PreservedAnalyses AFLCoverage::run(Module &M, ModuleAnalysisManager &MAM) {

#else
bool AFLCoverage::runOnModule(Module &M) {

#endif

  LLVMContext &C = M.getContext(); // get the context of M

  IntegerType *Int8Ty = IntegerType::getInt8Ty(C); // define i8 type
  IntegerType *Int32Ty = IntegerType::getInt32Ty(C); // define i32 type
#ifdef AFL_HAVE_VECTOR_INTRINSICS // "vectorize instrumentation" flag
  IntegerType *IntLocTy = // if AFL_HAVE_VECTOR_INTRINSICS enables, define IntLocTy type to store prev_loc
      IntegerType::getIntNTy(C, sizeof(PREV_LOC_T) * CHAR_BIT);
#endif
  struct timeval  tv; // time value for now
  struct timezone tz; // time zone for now
  u32             rand_seed;
  unsigned int    cur_loc = 0; // current instrumented location ID

  /* Setup random() so we get Actually Random(TM) outputs from AFL_R() */
  gettimeofday(&tv, &tz); // get current time of the day
  rand_seed = tv.tv_sec ^ tv.tv_usec ^ getpid(); // xor with process pid to get rand seed
  AFL_SR(rand_seed); // marcro to set rand seed (later uses AFL_R() to generate random number)

  /* Show a banner */

  setvbuf(stdout, NULL, _IONBF, 0); // set no buffer of IO

  if (getenv("AFL_DEBUG")) debug = 1; // if AFL_DEBUG enables then set debug = 1

#if LLVM_VERSION_MAJOR >= 11                        /* use new pass manager */
  if (getenv("AFL_SAN_NO_INST")) { // if AFL_SAN_NO_INST enables, then skip instrumentation

    if (debug) { fprintf(stderr, "Instrument disabled\n"); }
    return PreservedAnalyses::all();

  }

#else
  if (getenv("AFL_SAN_NO_INST")) { // if AFL_SAN_NO_INST enables, then skip instrumentation

    if (debug) { fprintf(stderr, "Instrument disabled\n"); }
    return true;

  }

#endif

...
...
...

#if LLVM_VERSION_MAJOR >= 11                        /* use new pass manager */
  return PreservedAnalyses(); // LLVM-11+ uses PreservedAnalyses to show IR modified
#else
  return true; // LLVM-10- uses true to show IR modified
#endif

}
```



下面逐部分解释代码

主逻辑进来设置参数以后, 开始设置插桩率 `inst_ratio`, `neverzero counters`, `threadsafe counters`. 接着, 设置上下文插桩相关的变量, 包括 `N-gram`, `ctx`, `caller`. 

```c
  // if fd==2(stderr) && AFL_QUIET disables && AFL_DEBUG enables then output the version information 
  if ((isatty(2) && !getenv("AFL_QUIET")) || getenv("AFL_DEBUG") != NULL) {

    SAYF(cCYA "afl-llvm-pass" VERSION cRST
              " by <lszekeres@google.com> and <adrian.herrera@anu.edu.au>\n");

  } else // else set the flag be_quiet = 1

    be_quiet = 1;

  // ...

  /* Decide instrumentation ratio */

  char        *inst_ratio_str = getenv("AFL_INST_RATIO"); // use AFL_INST_RATIO to set instrumentation ratio
  unsigned int inst_ratio = 100; // 100% as max

  if (inst_ratio_str) {
	// if inst_ratio_str is not numeric || inst_ratio == 0 || inst_ratio > 100 then report error and exit
    if (sscanf(inst_ratio_str, "%u", &inst_ratio) != 1 || !inst_ratio ||
        inst_ratio > 100)
      FATAL("Bad value of AFL_INST_RATIO (must be between 1 and 100)");

  }

#if LLVM_VERSION_MAJOR < 9
  char *neverZero_counters_str = getenv("AFL_LLVM_NOT_ZERO"); // LLVM-8- set AFL_LLVM_NOT_ZERO to disable neverZero counters
#endif
  skip_nozero = getenv("AFL_LLVM_SKIP_NEVERZERO"); // LLVM-9+ set AFL_LLVM_SKIP_NEVERZERO to disable neverZero counters
  use_threadsafe_counters = getenv("AFL_LLVM_THREADSAFE_INST"); // set AFL_LLVM_THREADSAFE_INST to enable threadsafe counters

  if ((isatty(2) && !getenv("AFL_QUIET")) || !!getenv("AFL_DEBUG")) {

    if (use_threadsafe_counters) { // when enables threadsafe counters then skip neverzero counters

      skip_nozero = "1";
      SAYF(cCYA "afl-llvm-pass" VERSION cRST " using thread safe counters\n");

    } else {

      SAYF(cCYA "afl-llvm-pass" VERSION cRST
                " using non-thread safe instrumentation\n");

    }

  }

  unsigned PrevLocSize = 0;
  unsigned PrevCallerSize = 0;

  char *ngram_size_str = getenv("AFL_LLVM_NGRAM_SIZE"); // set ngram_size by AFL_LLVM_NGRAM_SIZE 
  if (!ngram_size_str) ngram_size_str = getenv("AFL_NGRAM_SIZE"); // or set ngram_size by AFL_NGRAM_SIZE
  char *ctx_k_str = getenv("AFL_LLVM_CTX_K"); // enables context-aware instrumentaion size by setting AFL_LLVM_CTX_K
  if (!ctx_k_str) ctx_k_str = getenv("AFL_CTX_K"); // or enables context-aware instrumentaion size by setting AFL_CTX_K
  ctx_str = getenv("AFL_LLVM_CTX"); // enables instrument_ctx by setting AFL_LLVM_CTX
  caller_str = getenv("AFL_LLVM_CALLER"); // or enables instrument_ctx by setting AFL_LLVM_CALLER

  bool instrument_ctx = ctx_str || caller_str; // set instrument_ctx to enable context-aware instrumentation
```



往下是设置向量化插桩的参数, 包括 `n-gram`, `k-context`. 

```c
#ifdef AFL_HAVE_VECTOR_INTRINSICS // AFL_HAVE_VECTOR_INTRINSICS to enable vectorize instrumentation, e.g. N-gram path log
  /* Decide previous location vector size (must be a power of two) */
  VectorType *PrevLocTy = NULL;

  if (ngram_size_str)
    // if ngram_size is not numeric || ngram_size <= 1 || ngram_size > allowed NGRAM_SIZE_MAX then report error and exit
    if (sscanf(ngram_size_str, "%u", &ngram_size) != 1 || ngram_size < 2 ||
        ngram_size > NGRAM_SIZE_MAX)
      FATAL(
          "Bad value of AFL_NGRAM_SIZE (must be between 2 and NGRAM_SIZE_MAX "
          "(%u))",
          NGRAM_SIZE_MAX);
  // ngram_size == 1 makes no sense to record only 1 path, so set to 0
  if (ngram_size == 1) ngram_size = 0;
  if (ngram_size)
    PrevLocSize = ngram_size - 1; // PrevLoc to store ngram_size - 1 path log
  else
    PrevLocSize = 1; // default is 1

  /* Decide K-ctx vector size (must be a power of two) */
  VectorType *PrevCallerTy = NULL;

  if (ctx_k_str)
    // check valid ctx_k_str setting
    if (sscanf(ctx_k_str, "%u", &ctx_k) != 1 || ctx_k < 1 || ctx_k > CTX_MAX_K)
      FATAL("Bad value of AFL_CTX_K (must be between 1 and CTX_MAX_K (%u))",
            CTX_MAX_K);

  if (ctx_k == 1) { // when ctx_k == 1 then only enables CALLER mode

    ctx_k = 0;
    instrument_ctx = true;
    caller_str = ctx_k_str;  // Enable CALLER instead

  }

  if (ctx_k) { // when ctx_k >= 2 then enables context instrumentation

    PrevCallerSize = ctx_k; // set the PrevCallerSize as k
    instrument_ctx = true;

  }

#else // disables context instrumentation
  if (ngram_size_str) // report error when setting ngram_size but not setting AFL_HAVE_VECTOR_INTRINSICS
  #ifndef LLVM_VERSION_PATCH
    FATAL(
        "Sorry, NGRAM branch coverage is not supported with llvm version "
        "%d.%d.%d!",
        LLVM_VERSION_MAJOR, LLVM_VERSION_MINOR, 0);
  #else
    FATAL(
        "Sorry, NGRAM branch coverage is not supported with llvm version "
        "%d.%d.%d!",
        LLVM_VERSION_MAJOR, LLVM_VERSION_MINOR, LLVM_VERSION_PATCH);
  #endif
  if (ctx_k_str) // report error when setting ctx_k but not setting AFL_HAVE_VECTOR_INTRINSICS
  #ifndef LLVM_VERSION_PATCH
    FATAL(
        "Sorry, K-CTX branch coverage is not supported with llvm version "
        "%d.%d.%d!",
        LLVM_VERSION_MAJOR, LLVM_VERSION_MINOR, 0);
  #else
    FATAL(
        "Sorry, K-CTX branch coverage is not supported with llvm version "
        "%d.%d.%d!",
        LLVM_VERSION_MAJOR, LLVM_VERSION_MINOR, LLVM_VERSION_PATCH);
  #endif
  PrevLocSize = 1; // default PrevLocSize is 1
#endif

#ifdef AFL_HAVE_VECTOR_INTRINSICS
  int PrevLocVecSize = PowerOf2Ceil(PrevLocSize); // PrevLocSize <= 2^n (where n is the smallest to make inequality hold)
  if (ngram_size)
    PrevLocTy = VectorType::get(IntLocTy, PrevLocVecSize 
                                // create vector PrevLocTy[PrevLocVecSize] whose element type is IntLocTy
  #if LLVM_VERSION_MAJOR >= 12 // LLVM-12+ supports scalable vector
                                ,
                                false // not scalable to expend vector
  #endif
    );
#endif

#ifdef AFL_HAVE_VECTOR_INTRINSICS
  int PrevCallerVecSize = PowerOf2Ceil(PrevCallerSize); 
  // PrevCallerSize <= 2^n (where n is the smallest to make inequality hold)
  if (ctx_k)
    PrevCallerTy = VectorType::get(IntLocTy, PrevCallerVecSize
                                   // create vector PrevCallerTy[PrevCallerVecSize] whose element type is IntLocTy
  #if LLVM_VERSION_MAJOR >= 12 // LLVM-12+ supports scalable vector
                                   ,
                                   false // not scalable to expend vector
  #endif
    );
#endif
```



设置完上下文相关参数后, 接着是设置共享内存 `SHM` 的相关变量

```c
  /* Get globals for the SHM region and the previous location. Note that
     __afl_prev_loc is thread-local. */
  // set the global variable __afl_area_ptr, whose type is i8* (Int8Ty pointer).
  GlobalVariable *AFLMapPtr = // points to bitmap share memory
      new GlobalVariable(M, PointerType::get(Int8Ty, 0), false,
                         GlobalValue::ExternalLinkage, 0, "__afl_area_ptr");
  GlobalVariable *AFLPrevLoc; // last basic block location
  GlobalVariable *AFLPrevCaller; // last caller location
  GlobalVariable *AFLContext = NULL; // last function ID (caller mode enabling), which is thread local storage (TLS)

  if (ctx_str || caller_str) // if enables context instrumentation then create global variable __afl_prev_ctx
#if defined(__ANDROID__) || defined(__HAIKU__) || defined(NO_TLS) // ANDROID not supports TLS
    AFLContext = new GlobalVariable(
        M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_ctx");
#else
    AFLContext = new GlobalVariable(
        M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_ctx", 0,
        GlobalVariable::GeneralDynamicTLSModel, 0, false);
#endif

#ifdef AFL_HAVE_VECTOR_INTRINSICS
  if (ngram_size) // if enables ngram then create global variable __afl_prev_loc to store last basic block
  #if defined(__ANDROID__) || defined(__HAIKU__) || defined(NO_TLS)
    AFLPrevLoc = new GlobalVariable(
        M, PrevLocTy, /* isConstant */ false, GlobalValue::ExternalLinkage,
        /* Initializer */ nullptr, "__afl_prev_loc");
  #else
    AFLPrevLoc = new GlobalVariable(
        M, PrevLocTy, /* isConstant */ false, GlobalValue::ExternalLinkage,
        /* Initializer */ nullptr, "__afl_prev_loc",
        /* InsertBefore */ nullptr, GlobalVariable::GeneralDynamicTLSModel,
        /* AddressSpace */ 0, /* IsExternallyInitialized */ false);
  #endif
  else // disables vectorize mode so create not-vector global variable __afl_prev_loc
#endif
#if defined(__ANDROID__) || defined(__HAIKU__) || defined(NO_TLS)
    AFLPrevLoc = new GlobalVariable(
        M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_loc");
#else
  AFLPrevLoc = new GlobalVariable(
      M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_loc", 0,
      GlobalVariable::GeneralDynamicTLSModel, 0, false);
#endif

#ifdef AFL_HAVE_VECTOR_INTRINSICS
  if (ctx_k) // if enables caller then create global variable __afl_prev_caller to store last caller
  #if defined(__ANDROID__) || defined(__HAIKU__) || defined(NO_TLS)
    AFLPrevCaller = new GlobalVariable(
        M, PrevCallerTy, /* isConstant */ false, GlobalValue::ExternalLinkage,
        /* Initializer */ nullptr, "__afl_prev_caller");
  #else
    AFLPrevCaller = new GlobalVariable(
        M, PrevCallerTy, /* isConstant */ false, GlobalValue::ExternalLinkage,
        /* Initializer */ nullptr, "__afl_prev_caller",
        /* InsertBefore */ nullptr, GlobalVariable::GeneralDynamicTLSModel,
        /* AddressSpace */ 0, /* IsExternallyInitialized */ false);
  #endif
  else // disables vectorize mode so create not-vector global variable __afl_prev_caller
#endif
#if defined(__ANDROID__) || defined(__HAIKU__) || defined(NO_TLS)
    AFLPrevCaller =
        new GlobalVariable(M, Int32Ty, false, GlobalValue::ExternalLinkage, 0,
                           "__afl_prev_caller");
#else
  AFLPrevCaller = new GlobalVariable(
      M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_caller",
      0, GlobalVariable::GeneralDynamicTLSModel, 0, false);
#endif

#ifdef AFL_HAVE_VECTOR_INTRINSICS
  /* Create the vector shuffle mask for updating the previous block history.
     Note that the first element of the vector will store cur_loc, so just set
     it to undef to allow the optimizer to do its thing. */
  
  SmallVector<Constant *, 32> PrevLocShuffle = {UndefValue::get(Int32Ty)};
  
  for (unsigned I = 0; I < PrevLocSize - 1; ++I)
    PrevLocShuffle.push_back(ConstantInt::get(Int32Ty, I));
    // PrevLocShuffle = [cur_loc(UnderValue), prev_loc[0], prev_loc[1], ...]

  // fill remains of vector elements with PrevLocSize
  for (int I = PrevLocSize; I < PrevLocVecSize; ++I)
    PrevLocShuffle.push_back(ConstantInt::get(Int32Ty, PrevLocSize));

  // The mask will right shift the vector of PrevLoc and insert cur_loc into first place of vector
  Constant *PrevLocShuffleMask = ConstantVector::get(PrevLocShuffle);

  // same logic for PrevCallerShuffleMask 
  Constant                   *PrevCallerShuffleMask = NULL;
  SmallVector<Constant *, 32> PrevCallerShuffle = {UndefValue::get(Int32Ty)};

  if (ctx_k) {

    for (unsigned I = 0; I < PrevCallerSize - 1; ++I)
      PrevCallerShuffle.push_back(ConstantInt::get(Int32Ty, I));
      // PrevCallerShuffle = [cur_caller(UnderValue), prev_caller[0], prev_caller[1], ...]
    
    // fill remains of vector elements with PrevCallerSize
    for (int I = PrevCallerSize; I < PrevCallerVecSize; ++I)
      PrevCallerShuffle.push_back(ConstantInt::get(Int32Ty, PrevCallerSize));

    // The mask will right shift the vector of PrevCaller and insert cur_caller into first place of vector
    PrevCallerShuffleMask = ConstantVector::get(PrevCallerShuffle);

  }

#endif

  // other constants we need
  ConstantInt *One = ConstantInt::get(Int8Ty, 1);

  Value    *PrevCtx = NULL;     // CTX sensitive coverage
  LoadInst *PrevCaller = NULL;  // K-CTX coverage
```



**插桩的主要逻辑**

```c
  /* Instrument all the things! */
  // ----------> main logic of instrumentation starts here <----------
  int inst_blocks = 0; // count of successfully instrumented blocks
  scanForDangerousFunctions(&M); // scan M contains dangerous functions like fork, exec

  for (auto &F : M) { // traverse all functions in Module M

    int has_calls = 0;
    if (debug)
      fprintf(stderr, "FUNCTION: %s (%zu)\n", F.getName().str().c_str(),
              F.size());

    if (!isInInstrumentList(&F, MNAME)) { continue; } // skip function doesn't need instrument

    if (F.size() < function_minimum_size) { continue; } // skip function with little basic blocks 

    std::list<Value *> todo;
    for (auto &BB : F) { // traverse all basic blocks in Function F

      BasicBlock::iterator IP = BB.getFirstInsertionPt(); // IP points to first insertion point in BB
      IRBuilder<>          IRB(&(*IP)); // IRBuilder inserts IR instruction

      // Context sensitive coverage
      // if enables instrument_ctx && current BB is the first BB of function F, then ...
      if (instrument_ctx && &BB == &F.getEntryBlock()) {

#ifdef AFL_HAVE_VECTOR_INTRINSICS
        if (ctx_k) {

          PrevCaller = IRB.CreateLoad( // CreateLoad(Type, Value*): load type data from Value* address
  #if LLVM_VERSION_MAJOR >= 14
              PrevCallerTy,
  #endif
              AFLPrevCaller);
          
          // insert nosanitize to tell A/M/TSan not to analyze the instruction
          PrevCaller->setMetadata(M.getMDKindID("nosanitize"),
                                  MDNode::get(C, None));
          PrevCtx = // CreateZExt(Value*, Type*): zero extend an integer to wider type integer, 
              		// high position padding with zeros
              IRB.CreateZExt(IRB.CreateXorReduce(PrevCaller), IRB.getInt32Ty());
				// CreateXorReduce(Value*): xor each element in vector Value* and return the xored result
        } else // ctx_k == 0

#endif
        {

          // load the context ID of the previous function and write to a
          // local variable on the stack
          LoadInst *PrevCtxLoad = IRB.CreateLoad( // PrevCtxLoad stores prev function context ID
#if LLVM_VERSION_MAJOR >= 14
              IRB.getInt32Ty(),
#endif
              AFLContext);
          
          // insert nosanitize to tell A/M/TSan not to analyze the instruction
          PrevCtxLoad->setMetadata(M.getMDKindID("nosanitize"),
                                   MDNode::get(C, None));
          PrevCtx = PrevCtxLoad;

        } // end of if(ctx_k) else 

        // does the function have calls? and is any of the calls larger than one basic block?
        for (auto &BB_2 : F) { // traverse all BB in function F again 
            // to check if F calls other functions needed to be instrumented

          if (has_calls) break; // already found calls then break
          for (auto &IN : BB_2) { // traverse instruction by instruction in function

            CallInst *callInst = nullptr;
            if ((callInst = dyn_cast<CallInst>(&IN))) { // safely transform &IN to CallInst type

              Function *Callee = callInst->getCalledFunction(); // get the callee of instruction
              if (!Callee || Callee->size() < function_minimum_size) 
                  // if callee is nullptr || called function with little BB, then skip
                continue;
              else { // found one called function needed to be instrumented

                has_calls = 1; // set the flag has_calls = 1
                break;

              }

            }

          }

        } // end of for (auto &BB_2 : F)

        // if yes we store a context ID for this function in the global var
        if (has_calls) {
		  // random another new context ID
          Value *NewCtx = ConstantInt::get(Int32Ty, AFL_R(map_size));
#ifdef AFL_HAVE_VECTOR_INTRINSICS
          if (ctx_k) {
			// uses mask vector to right shift the old vector 
            Value *ShuffledPrevCaller = IRB.CreateShuffleVector(
                PrevCaller, UndefValue::get(PrevCallerTy), // UndefValue vector only occupy the places here
                PrevCallerShuffleMask);
            // and insert NewCtx into first place of PrevCaller vector
            Value *UpdatedPrevCaller = IRB.CreateInsertElement(
                ShuffledPrevCaller, NewCtx, (uint64_t)0);

            StoreInst *Store = // CreateStore(Value *Val, Value *Ptr): store Val to Ptr 
                IRB.CreateStore(UpdatedPrevCaller, AFLPrevCaller); // update AFLPrevCaller with UpdatedPrevCaller
            // insert nosanitize to tell A/M/TSan not to analyze the instruction
            Store->setMetadata(M.getMDKindID("nosanitize"),
                               MDNode::get(C, None));

          } else // ctx_k == 0

#endif
          {
			// when disables context mode then xor PrevCtx with NewCtx 
            if (ctx_str) NewCtx = IRB.CreateXor(PrevCtx, NewCtx);
            StoreInst *StoreCtx = IRB.CreateStore(NewCtx, AFLContext); // update AFLContext with NewCtx
            // insert nosanitize to tell A/M/TSan not to analyze the instruction
            StoreCtx->setMetadata(M.getMDKindID("nosanitize"),
                                  MDNode::get(C, None));

          } // end of if else

        } // end of has_calls

      } // end of if (instrument_ctx && &BB == &F.getEntryBlock())

      if (AFL_R(100) >= inst_ratio) continue; // skip rest BB instrumentation if random >= inst_ratio
      
      /* Make up cur_loc */

      // cur_loc++;
      cur_loc = AFL_R(map_size); // random cur_loc for current BB from 0 to map_size - 1

/* ... */ // ignore this bug fix code
#if !(LLVM_VERSION_MAJOR == 6 && LLVM_VERSION_MINOR == 0) || !defined __linux__
      
	  // only instrument if this basic block is the destination of a previous
      // basic block that has multiple successors
      // this gets rid of ~5-10% of instrumentations that are unnecessary
      // result: a little more speed and less map pollution
      int more_than_one = -1; // flag: BB's pred_BB has more than one succ_BB  
      // fprintf(stderr, "BB %u: ", cur_loc);
      for (pred_iterator PI = pred_begin(&BB), E = pred_end(&BB); PI != E; ++PI) {

        BasicBlock *Pred = *PI;

        int count = 0; // count for succ_BB  
        if (more_than_one == -1) more_than_one = 0;
        // fprintf(stderr, " %p=>", Pred);

        for (succ_iterator SI = succ_begin(Pred), E = succ_end(Pred); SI != E; ++SI) {

          BasicBlock *Succ = *SI;

          if (Succ != NULL) count++;

        }

        if (count > 1) more_than_one = 1; // succ_BB count is more than one 

      }

      // fprintf(stderr, " == %d\n", more_than_one);
      if (F.size() > 1 && more_than_one != 1) {

        // in CTX mode we have to restore the original context for the caller -
        // she might be calling other functions which need the correct CTX
        if (instrument_ctx && has_calls) {

          Instruction *Inst = BB.getTerminator(); // get the last instruction of BB
          if (isa<ReturnInst>(Inst) || isa<ResumeInst>(Inst)) { 
              // only last inst is ret or resum type need to restore context

            IRBuilder<> Post_IRB(Inst); // define IRBuilder Post_IRB, which is right before Inst instruction

            StoreInst *RestoreCtx;
  #ifdef AFL_HAVE_VECTOR_INTRINSICS
            if (ctx_k) // restore the context ID to AFLPrevCaller when enables ctx_k
              RestoreCtx = IRB.CreateStore(PrevCaller, AFLPrevCaller);
            else // restore the context ID to AFLContext when disables ctx_k
  #endif
              RestoreCtx = Post_IRB.CreateStore(PrevCtx, AFLContext);
            // insert nosanitize to tell A/M/TSan not to analyze the instruction
            RestoreCtx->setMetadata(M.getMDKindID("nosanitize"),
                                    MDNode::get(C, None));

          }

        } // end of if (instrument_ctx && has_calls)

        continue; // skip instrumentation if pred_BB not ends with conditional jump
          // which means the current BB is not in branching path, so no need to instrument

      } // end of if (F.size() > 1 && more_than_one != 1)

#endif

      ConstantInt *CurLoc; // define constant CurLoc to store current BB location

#ifdef AFL_HAVE_VECTOR_INTRINSICS
      if (ngram_size)
        CurLoc = ConstantInt::get(IntLocTy, cur_loc); // when enables ngram then CurLoc type is IntLocTy
      else
#endif
        CurLoc = ConstantInt::get(Int32Ty, cur_loc); // when disables ngram CurLoc type is Int32Ty

      /* Load prev_loc */

      LoadInst *PrevLoc;

      if (ngram_size) {

        PrevLoc = IRB.CreateLoad(
#if LLVM_VERSION_MAJOR >= 14
            PrevLocTy, // if ngram enables then load PrevLoc as PrevLocTy type
#endif
            AFLPrevLoc);

      } else {

        PrevLoc = IRB.CreateLoad(
#if LLVM_VERSION_MAJOR >= 14
            IRB.getInt32Ty(), // if ngram disables then load PrevLoc as Int32Ty type
#endif
            AFLPrevLoc);

      }
	  // insert nosanitize to tell A/M/TSan not to analyze the instruction
      PrevLoc->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
      Value *PrevLocTrans;

#ifdef AFL_HAVE_VECTOR_INTRINSICS
      /* "For efficiency, we propose to hash the tuple as a key into the
         hit_count map as (prev_block_trans << 1) ^ curr_block_trans, where
         prev_block_trans = (block_trans_1 ^ ... ^ block_trans_(n-1)" */

      if (ngram_size)
        PrevLocTrans = // xor PrevLoc each element to introduce n gram BBs information
            IRB.CreateZExt(IRB.CreateXorReduce(PrevLoc), IRB.getInt32Ty());
      else
#endif
        PrevLocTrans = PrevLoc;

      if (instrument_ctx)
        PrevLocTrans = // xor PrevLocTrans with PrevCtx to introduce context-aware
            IRB.CreateZExt(IRB.CreateXor(PrevLocTrans, PrevCtx), Int32Ty);
      else
        PrevLocTrans = IRB.CreateZExt(PrevLocTrans, IRB.getInt32Ty());
```



插桩之后会更新 bitmap, 也就是目标程序与 fuzzer 的共享内存, 反馈信息由此传输

```c
      /* Load SHM pointer */

      LoadInst *MapPtr = IRB.CreateLoad(
#if LLVM_VERSION_MAJOR >= 14
          PointerType::get(Int8Ty, 0),
#endif
          AFLMapPtr); // load share memory ptr from AFLMapPtr with type i8*
      // insert nosanitize to tell A/M/TSan not to analyze the instruction
      MapPtr->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

      Value *MapPtrIdx; // share memory (bitmap) index ptr

#ifdef AFL_HAVE_VECTOR_INTRINSICS
      if (ngram_size)
        MapPtrIdx = IRB.CreateGEP( // CreateGEP(type, ptr1, ptr2): ptr1 + ptr2
            Int8Ty, MapPtr,
            IRB.CreateZExt(
                IRB.CreateXor(PrevLocTrans, IRB.CreateZExt(CurLoc, Int32Ty)),
                Int32Ty));
      else
#endif
        MapPtrIdx = IRB.CreateGEP(
#if LLVM_VERSION_MAJOR >= 14
            Int8Ty,
#endif
            MapPtr, IRB.CreateXor(PrevLocTrans, CurLoc));
	  // logic above is like this: MapPtrIdx = MapPtr + ((PrevLoc ^ CurLoc) & 0xFFFF)

	  /* Update bitmap */

      if (use_threadsafe_counters) {                              /* Atomic */
		// logic of function below is like: bitmap[MapPtrIdx] += 1;
        IRB.CreateAtomicRMW(llvm::AtomicRMWInst::BinOp::Add, MapPtrIdx, One,
#if LLVM_VERSION_MAJOR >= 13
                            llvm::MaybeAlign(1),
#endif
                            llvm::AtomicOrdering::Monotonic);
        /*

                }

        */

      } else { // not Atomic operations

        LoadInst *Counter = IRB.CreateLoad(
#if LLVM_VERSION_MAJOR >= 14
            IRB.getInt8Ty(),
#endif
            MapPtrIdx);
        Counter->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

        Value *Incr = IRB.CreateAdd(Counter, One); // Incr = bitmap[MapPtrIdx] + 1;

#if LLVM_VERSION_MAJOR >= 9
        if (!skip_nozero) {

#else
        if (neverZero_counters_str != NULL) { // if enables neverzero then skip 0 counter value

#endif
          /* hexcoder: Realize a counter that skips zero during overflow.
           * Once this counter reaches its maximum value, it next increments to
           * 1
           *
           * Instead of
           * Counter + 1 -> Counter
           * we inject now this
           * Counter + 1 -> {Counter, OverflowFlag}
           * Counter + OverflowFlag -> Counter
           */

          ConstantInt *Zero = ConstantInt::get(Int8Ty, 0); // get constant 0 with type i8
          auto         cf = IRB.CreateICmpEQ(Incr, Zero); // if Incr == 0, means overflow
          auto         carry = IRB.CreateZExt(cf, Int8Ty); // carry = 1 when Incr == 0
          Incr = IRB.CreateAdd(Incr, carry); // Incr plus carry to avoid 0 value

        }

        IRB.CreateStore(Incr, MapPtrIdx)
            ->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

      }                                                  /* non atomic case */

      /* Update prev_loc history vector (by placing cur_loc at the head of the
         vector and shuffle the other elements back by one) */

      StoreInst *Store;

#ifdef AFL_HAVE_VECTOR_INTRINSICS
      if (ngram_size) {

        Value *ShuffledPrevLoc = IRB.CreateShuffleVector(
            PrevLoc, UndefValue::get(PrevLocTy), PrevLocShuffleMask);
        Value *UpdatedPrevLoc = IRB.CreateInsertElement(
            ShuffledPrevLoc, IRB.CreateLShr(CurLoc, (uint64_t)1), (uint64_t)0);
			// CreateLShr(val1, val2): val1 >> val2  
        Store = IRB.CreateStore(UpdatedPrevLoc, AFLPrevLoc);
        Store->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

      } else

#endif
      {

        Store = IRB.CreateStore(ConstantInt::get(Int32Ty, cur_loc >> 1),
                                AFLPrevLoc);
        Store->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

      }

      // in CTX mode we have to restore the original context for the caller -
      // she might be calling other functions which need the correct CTX.
      // Currently this is only needed for the Ubuntu clang-6.0 bug
      if (instrument_ctx && has_calls) { // when enables context then restore the context of BB

        Instruction *Inst = BB.getTerminator();
        if (isa<ReturnInst>(Inst) || isa<ResumeInst>(Inst)) {

          IRBuilder<> Post_IRB(Inst);

          StoreInst *RestoreCtx;
#ifdef AFL_HAVE_VECTOR_INTRINSICS
          if (ctx_k)
            RestoreCtx = IRB.CreateStore(PrevCaller, AFLPrevCaller);
          else
#endif
            RestoreCtx = Post_IRB.CreateStore(PrevCtx, AFLContext);
          RestoreCtx->setMetadata(M.getMDKindID("nosanitize"),
                                  MDNode::get(C, None));

        }

      }

      inst_blocks++; // successful instrumentation count++ 

    } // end of if (instrument_ctx && has_calls)
```



最后, 是线程安全和计数器更新的新版代码, 主要逻辑是使用 CAS (Compare-And-Swap) 循环替代原本的 `++` 操作. 优点是并发更安全, 避免计数器为 0 的问题, 缺点是效率比 `++` 更低, 目前作为未来计划之一, 尚未启用. 

```c
#if 0
    if (use_threadsafe_counters) {                       /* Atomic NeverZero */
      // handle the list of registered blocks to instrument
      for (auto val : todo) {

        /* hexcoder: Realize a thread-safe counter that skips zero during
         * overflow. Once this counter reaches its maximum value, it next
         * increments to 1
         * ...
		 */

        Value *              MapPtrIdx = val;
        Instruction *        MapPtrIdxInst = cast<Instruction>(val);
        BasicBlock::iterator it0(&(*MapPtrIdxInst));
        ++it0;
        IRBuilder<> IRB(&(*it0));

        // load the old counter value atomically
        LoadInst *Counter = IRB.CreateLoad(
  #if LLVM_VERSION_MAJOR >= 14
        IRB.getInt8Ty(),
  #endif
        MapPtrIdx);
        Counter->setAlignment(llvm::Align());
        Counter->setAtomic(llvm::AtomicOrdering::Monotonic);
        Counter->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

        BasicBlock *BB = IRB.GetInsertBlock();
        // insert a basic block with the corpus of a do while loop
        // the calculation may need to repeat, if atomic compare_exchange is not
        // successful

        BasicBlock::iterator it(*Counter);
        it++;  // split after load counter
        BasicBlock *end_bb = BB->splitBasicBlock(it);
        end_bb->setName("injected");

        // insert the block before the second half of the split
        BasicBlock *do_while_bb =
            BasicBlock::Create(C, "injected", end_bb->getParent(), end_bb);

        // set terminator of BB from target end_bb to target do_while_bb
        auto term = BB->getTerminator();
        BranchInst::Create(do_while_bb, BB);
        term->eraseFromParent();

        // continue to fill instructions into the do_while loop
        IRB.SetInsertPoint(do_while_bb, do_while_bb->getFirstInsertionPt());

        PHINode *PN = IRB.CreatePHI(Int8Ty, 2);

        // compare with maximum value 0xff
        auto *Cmp = IRB.CreateICmpEQ(Counter, ConstantInt::get(Int8Ty, -1));

        // increment the counter
        Value *Incr = IRB.CreateAdd(Counter, One);

        // select the counter value or 1
        auto *Select = IRB.CreateSelect(Cmp, One, Incr);

        // try to save back the new counter value
        auto *CmpXchg = IRB.CreateAtomicCmpXchg(
            MapPtrIdx, PN, Select, llvm::AtomicOrdering::Monotonic,
            llvm::AtomicOrdering::Monotonic);
        CmpXchg->setAlignment(llvm::Align());
        CmpXchg->setWeak(true);
        CmpXchg->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

        // get the result of trying to update the Counter
        Value *Success =
            IRB.CreateExtractValue(CmpXchg, ArrayRef<unsigned>({1}));
        // get the (possibly updated) value of Counter
        Value *OldVal =
            IRB.CreateExtractValue(CmpXchg, ArrayRef<unsigned>({0}));

        // initially we use Counter
        PN->addIncoming(Counter, BB);
        // on retry, we use the updated value
        PN->addIncoming(OldVal, do_while_bb);

        // if the cmpXchg was not successful, retry
        IRB.CreateCondBr(Success, end_bb, do_while_bb);

      }

    }

#endif // end of #if 0

  } // end of for (auto &F : M)

  /*
    // This is currently disabled because we not only need to create/insert a
    // function (easy), but also add it as a constructor with an ID < 5
    ...
  */

  /* Say something nice. */

  if (!be_quiet) { // print out information for debugging

    if (!inst_blocks)
      WARNF("No instrumentation targets found.");
    else {

      char modeline[100];
      snprintf(modeline, sizeof(modeline), "%s%s%s%s%s%s",
               getenv("AFL_HARDEN") ? "hardened" : "non-hardened",
               getenv("AFL_USE_ASAN") ? ", ASAN" : "",
               getenv("AFL_USE_MSAN") ? ", MSAN" : "",
               getenv("AFL_USE_CFISAN") ? ", CFISAN" : "",
               getenv("AFL_USE_TSAN") ? ", TSAN" : "",
               getenv("AFL_USE_UBSAN") ? ", UBSAN" : "");
      OKF("Instrumented %d locations (%s mode, ratio %u%%).", inst_blocks,
          modeline, inst_ratio);

    }

  }
```





## Reference

[1] https://github.com/AFLplusplus/AFLplusplus

[2] https://nyx-fuzz.com/papers/redqueen.pdf

