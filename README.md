# 編譯Gem5
```
scons EXTRAS=../NVmain build/X86/gem5.opt
```
# (Q2) Enable L3 last level cache in GEM5 + NVMAIN 
修改Cache.py，將L2Cache複製一份並改成L3Cache
```python
class L3Cache(Cache):
    assoc = 8
    tag_latency = 20
    data_latency = 20
    response_latency = 20
    mshrs = 20
    tgts_per_mshr = 12
    write_buffers = 8
```
修改CacheConfig.py，將下面這段並增加l3_cache_class
```python
dcache_class, icache_class, l2_cache_class, walk_cache_class = \
            O3_ARM_v7a_DCache, O3_ARM_v7a_ICache, O3_ARM_v7aL2, \
	    O3_ARM_v7aWalkCache, O3_ARM_v7aL3
```
修改後
```python
dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
            O3_ARM_v7a_DCache, O3_ARM_v7a_ICache, O3_ARM_v7aL2, \
	    O3_ARM_v7aWalkCache, O3_ARM_v7aL3
```
在剛才那段的正下方找到下面程式碼並增加L3Cache與l3_cache_class
```python
else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class = \
            L1_DCache, L1_ICache, L2Cache,None, L3Cache
```
修改後
```python
else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
            L1_DCache, L1_ICache, L2Cache,None, L3Cache
```
同樣在CacheConfig.py找到下面這段並增加options.l3cache
```python
if options.l2cache:
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)
	system.l3 = l3_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l3_size,
                                   assoc=options.l3_assoc)

        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
	system.tol3bus = L3XBar(clk_domain = system.cpu_clk_domain)

        system.l2.cpu_side = system.tol2bus.master
        system.l2.mem_side = system.tol3bus.slave

        system.l3.cpu_side = system.tol3bus.master
        system.l3.mem_side = system.membus.slave
```
修改後
```python
if options.l2cache and  options.l3cache:
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)
	system.l3 = l3_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l3_size,
                                   assoc=options.l3_assoc)

        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
	system.tol3bus = L3XBar(clk_domain = system.cpu_clk_domain)

        system.l2.cpu_side = system.tol2bus.master
        system.l2.mem_side = system.tol3bus.slave

        system.l3.cpu_side = system.tol3bus.master
        system.l3.mem_side = system.membus.slave
```
修改XBar.py，將L2XBar複製一份並改成L3Xbar
```python
class L3XBar(CoherentXBar):
    # 256-bit crossbar by default
    width = 32

    # Assume that most of this is covered by the cache latencies, with
    # no more than a single pipeline stage for any packet.
    frontend_latency = 1
    forward_latency = 0
    response_latency = 1
    snoop_response_latency = 1

    # Use a snoop-filter by default, and set the latency to zero as
    # the lookup is assumed to overlap with the frontend latency of
    # the crossbar
    snoop_filter = SnoopFilter(lookup_latency = 0)

    # This specialisation of the coherent crossbar is to be considered
    # the point of unification, it connects the dcache and the icache
    # to the first level of unified cache.
    point_of_unification = True
```
在BaseCPU.py中import剛才修改的L3XBar
```python
from XBar import L3XBar
```
修改BaseCPU.py，在class BaseCPU中加入
```python
def addThreeLevelCacheHierarchy(self, ic, dc, l3c, iwc = None, dwc = None):
        self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
        self.toL3Bus = L3XBar()
        self.connectCachedPorts(self.toL3Bus)
        self.l3cache = l3c
        self.toL2Bus.master = self.l3cache.cpu_side
        self._cached_ports = ['l3cache.mem_side']
```
修改Option.py，將下面程式碼加到# Cache Options下
```python
parser.add_option("--l3cache", action="store_true")
```
# (Q3) Config last level cache to 2-way and full-way associative cache and test performance

使用指令編譯./quicksort.c產生./quicksort
```
gcc --static quicksort.c -o quicksort
```
**因為將array大小設成100000效果不佳，所以更改成1000000**


Config last level cache to 2-way associative cache and test performance
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
Config last level cache to full-way associative cache and test performance(--l3_assoc改成1)
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=1 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
Config last level cache to full-way associative cache and test performance(--l3_assoc改成16384)
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=16384 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
因為在Options.py中有寫到
```
parser.add_option("--cacheline_size", type="int", default=64)
```
所以計算出有16384個block


# (Q4) Modify last level cache policy based on frequency based replacement policy
在Cache.py的L3Cache中增加
```python
replacement_policy = Param.BaseReplacementPolicy(LFURP(),"Replacement policy")
```
執行quicksort
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
# (Q5) Test the performance of write back and write through policy based on 4-way associative cache with isscc_pcm
在./src/mem/cache/base.cc找到下面這段並修改
```C
else if (blk && (pkt->needsWritable() ? blk->isWritable() :
                       blk->isReadable())) {
        // OK to satisfy access
        incHitCount(pkt);
        satisfyRequest(pkt, blk);
        maintainClusivity(pkt->fromCache(), blk);

        return true;
    }
```
修改後
```C
else if (blk && (pkt->needsWritable() ? blk->isWritable() :
                       blk->isReadable())) {
        // OK to satisfy access
        incHitCount(pkt);
        satisfyRequest(pkt, blk);
        maintainClusivity(pkt->fromCache(), blk);
	
        if (blk ->isWritable()){
	    PacketPtr  writeCleanPkt = writecleanBlk(blk, pkt->req->getDest(), pkt->id);
	    writebacks.push_back(writeCleanPkt);
	}

        return true;
    }
```
跑multiply
```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=4 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
