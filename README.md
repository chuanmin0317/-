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
在CacheConfig.py找到下面這段並增加l3_cache_class
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
        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
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
# Config last level cache to 2-way associative cache and test performance
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
# Config last level cache to full-way associative cache and test performance
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=16384 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config

```
