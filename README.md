NVmain + gem5 Simulation
===

**問題集**
---

* GEM5 + NVMAIN BUILD-UP (40%)
* Enable L3 last level cache in GEM5 + NVMAIN (15%)
* Config last level cache to  2-way and full-way associative cache and test performance (15%)
    *必須跑benchmark quicksort在 2-way跟 full way*
* Modify last level cache policy based on frequency based replacement policy (15%)
* Test the performance of write back and write through policy based on 4-way associative cache with isscc_pcm(15%) 
    *必須跑 benchmark multiply 在 write through跟 write back*
* Bonus (10%)
    Design last level cache policy to reduce the energy consumption of pcm_based main memory 
    Baseline:LRU


**1. 環境建置**
---

用ubuntu 18.04的vm或container都行，我是使用docker container，架設上較比較不占host server的資源，加上我不喜歡vmware的介面，容易卡頓等等。

此project模擬使用[gem5 Timing Simple CPU](https://www.gem5.org/documentation/general_docs/cpu_models/SimpleCPU)

git clone之後先進去資料夾中並把container給跑起來
```
$ cd Computer-Organization-Final-Project/
$ sudo docker-compose up -d --build
```
之後進去container當中
```
$ sudo docker exec -u root -it ubuntu_1804 bash
```
依序執行以下的指令
```
$ apt install build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python3-dev python3-six python libboost-all-dev pkg-config
$ git clone https://gem5.googlesource.com/public/gem5/
$ cd gem5
$ git checkout 525ce650e1a5bbe
$ git checkout -b gem5_good
```
到這裡就完成了一開始的設定 接著按照ppt開始編譯gem5並將NVmain給git clone進此專案就可以了

**Enable L3 last level cache**
---

共需要修改五個檔案

* gem5/configs/common/Caches.py
    在class L2Cache之後加入以下內容
    想了解object在gem5裡面的定義與使用等等可以參考[gem5:creating a simple config file and script](https://www.gem5.org/documentation/learning_gem5/part1/simple_config/)
    裡面有解說gem5如何進行模擬與怎麼用物件搭出cpu架構
```python=
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency = 20
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16
```
* gem5/configs/common/CacheConfig.py
    有兩個地方要修改
    1. 修改此檔案中的config_cache() function，將if options.cpu_type=="O3_ARM_v7a_3"底下修改成
    ```python=
    if options.cpu_type == "O3_ARM_v7a_3":
        try:
            from cores.arm.O3_ARM_v7a import *
        except:
            print("O3_ARM_v7a_3 is unavailable. Did you compile the O3 model?")
            sys.exit(1)

        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
            O3_ARM_v7a_DCache, O3_ARM_v7a_ICache, O3_ARM_v7aL2, \
            O3_ARM_v7aWalkCache,O3_ARM_v7aL3
    else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class= \
            L1_DCache, L1_ICache, L2Cache, None, L3Cache

        if buildEnv['TARGET_ISA'] == 'x86':
            walk_cache_class = PageTableWalkerCache
    ```
    2. 將if options.l2cache替換成以下
    ```python=
        if options.l2cache and options.l3cache:
        system.l2=l2_cache_class(
            clk_domain=system.cpu_clk_domain,
            size=options.l2_size,
            assoc=options.l2_assoc
        )
        system.l3=l3_cache_class(
            clk_domain=system.cpu_clk_domain,
            size=options.l3_size,
            assoc=options.l3_assoc
        )

        system.tol2bus=L2XBar(clk_domain=system.cpu_clk_domain)
        system.tol3bus=L3XBar(clk_domain=system.cpu_clk_domain)

        system.l2.cpu_side=system.tol2bus.master
        system.l2.mem_side=system.tol3bus.slave
        system.l3.cpu_side=system.tol3bus.master
        system.l3.mem_side=system.membus.slave
    
    elif operations.l2cache:
        # Provide a clock for the L2 and the L1-to-L2 bus here as they
        # are not connected using addTwoLevelCacheHierarchy. Use the
        # same clock as the CPUs.
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)

        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
        system.l2.cpu_side = system.tol2bus.master
        system.l2.mem_side = system.membus.slave
    ```

* gem5/src/mem/XBar.py
    新增一個class L3XBar
```python=
class L3XBar(CoherentXBar):
    width=32

    frontend_latency=1
    forward_latency=0
    response_latency=1
    snoop_response_latency=1
```
* gem5/src/cpu/BaseCPU.py
    從XBar.py導入L3XBar
    `from XBar import L3XBar`
    並且在BaseCPU這個class底下新增一個function
```python=
    def addThreeLevelCacheHierarchy(self, ic, dc, l3c, iwc=None, dwc=None):
        self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
        self.toL3bus=L3XBar()
        self.connectCachedPorts(self.toL3Bus)
        self.l3cache=l3c
        self.toL2Bus.master=self.l3cache.cpu_side
        self._cached_ports=['l3cache.mem_side']
```
* gem5/configs/common/Options.py
    在class addNoISAOptions底下新增一個參數，來啟用l3 cache
```python=
parser.add_option("--l3cache", action="store_true")
```

全部修改完後重新編譯一次(記得用root進入container裏頭)，在gem5目錄底下
```
scons EXTRAS=../NVmain build/X86/gem5.opt
```
編譯成功後使用
```
./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
成功之後可以用cat來查看l3關鍵字，有找到應該就是有成功
```
cat ./m5out/stats.txt | grep l3
```

**Config last level cache to 2-way and full-way associative cache and test performance**
---

查看performance的方式可以參考:[Understanding gem5 statistics and output](https://www.gem5.org/documentation/learning_gem5/part1/gem5_stats/)

主要是要查看stats.txt裏頭的host_inst_rate
在甚麼都還沒改動的時候(啟動l3 cache後)查看應該可以看到
```
# cat ./m5out/stats.txt | grep host_inst_rate
host_inst_rate                                 146110                       # Simulator instruction rate (inst/s)
```

修改associativity從command line給不同參數就可以
```
--<cache_name>_size=<size> 可以改變cahce size，例如
--l3_size=1MB

--<cache>_assoc=<way> 可以用來指定n-way associative
--l2_assoc=8
```

2-way set associative的指令如下
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config   
```

full-way associative的指令如下
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=1 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

為何fully associative是把associativity設成1?
這樣不是direct mapped的意思嗎
Answer: http://old.gem5.org/Coherence-Protocol-Independent_Memory_Components.html
看CacheMemory的model介紹那邊有寫到

由於助教把quicksort.c裏頭的array size開太大，這部分可能會跑很久，可以先把array size調小來驗證參數

**Modify last level cache policy based on frequency based replacement policy**
---

將Caches.py當中class L3Cache部分加入以下的部分即可
```python=
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency = 20
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16

    replacement_policy = Param.BaseReplacementPolicy(RRIPRP(),"Replacement policy")
```

之後檢查config.ini當中system.l3.replacement_policy的btp是否等於0
等於0就是設定成功了

https://www.gushiciku.cn/pl/2igF/zh-tw
https://medium.com/@arpitguptarag/adaptive-insertion-policies-for-high-performance-caching-a741c52f515c
bimodal throttle parameter,btp (ϵ), which controls the percentage of incoming lines that are placed in the MRU position.

**Test the performance of write back and write through policy based on 4-way associative cache with isscc_pcm**
---

研究gem5的source code...
本來以為要自己寫出這些mechanism,看起來是有幫我們寫好不過要了解怎麼使用
https://pages.cs.wisc.edu/~swilson/gem5-docs/classBaseCache.html


Packet定義 (在packet.hh第245行)
> A Packet is used to encapsulate a transfer between two objects in the memory system

**Important functions**
void BaseCache::allocateWriteBuffer	(	PacketPtr 	pkt,Tick 	time )
Cache::doWritebacks()
Cache::recvTimingResp()
Cache::recvTimingReq()

看完文件gem5預設就是一路write back到最底下，所以不用更改任何東西就是write back
https://pages.cs.wisc.edu/~swilson/gem5-docs/classBaseCache.html

BaseCache::writecleanBlk(CacheBlk *blk, Request::Flags dest, PacketId id)
裡面的如果dest is true就會set write through

看packet.hh裡面setWriteThrough()的definition
packet.hh裡面已經把write through需要用到的function寫好了

嘗試把dest強迫設1

套用base.cc第577行的作法
```c=
    if (pkt->isClean() && blk && blk->isDirty()) {
        // A cache clean opearation is looking for a dirty
        // block. If a dirty block is encountered a WriteClean
        // will update any copies to the path to the memory
        // until the point of reference.
        DPRINTF(CacheVerbose, "%s: packet %s found block: %s\n",
                __func__, pkt->print(), blk->print());
        PacketPtr wb_pkt = writecleanBlk(blk, pkt->req->getDest(), pkt->id);
        writebacks.push_back(wb_pkt);
        pkt->setSatisfied();
    }
```

可以知道if裡面的code可以將該block一直更新至memory
套用相同的做法在access裡面
在1073行加入以下的code
```cpp=
        if (blk->isWritable()) {
            PacketPtr writeclean_pkt = writecleanBlk(blk, pkt->req->getDest(), pkt->id);
            writebacks.push_back(writeclean_pkt);
        }
```

```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=4 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```