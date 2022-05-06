# Recreating Revice: CSCE 614 Term Project
This term project for CSCE 614: Computer Architecture is an attempt to recreate the solution to Spectre and Meltdown attacls as presented in *ReViCe: Reusing Victim Cache to Prevent Speculative Cache Leakage* by Kim et al. (2020).

## Team Members

 - Matan Broner
 - Abhiyash Hodge

## Main Code Modifications

### `gem5/src/cpu/o3/lsq.cc`

 - Added a new `isSpec` argument to the class constructor for `LSQRequest`. This is applied to both child classes `SingleDataRequest` and `SplitDataRequest`.
 - Modified the `buildPackets()` method of both `LSQRequest` sub-types to call the `Packet:createReadSpec()` method instead of `Packet:createRead()` if the incoming instruction is speculative.

### `gem5/src/mem/packet.hh`
- Added new enum `SpecState: {IS_SPEC, IS_NOT_SPEC}` and matching `_specState` class attribute.
- Added new enum `SpecIssueState: {ISSUED, SQUASHED, COMMITED}` and matching `_specIssueState` class attribute.
- Added methods `isSpecLoad()`, `isSpecSquashed()`, `isSpecCommitted()`, and `isSpecIssued()`.
- Added method `makeReadSpecCmd()`.

### `gem5/src/mem/port.hh`
- Added method `sendSpecLoadUpdateReq()`

### 	`gem5/src/mem/protocol/timing.hh`
- Added method `TimingRequestProtocol:sendSpecLoadUpdateReq()`
- Added method `TimingResponseProtocol:sendSpecLoadUpdateReq()`

### `gem5/src/mem/ruby/protocol/MESI_Two_Level-L1cache.sm`
- Set new `public="yes"` flag to method `getL1DCacheEntry()` (ie. custom compiler flag we added)

### `gem5/src/mem/ruby/system/RubyPort.cc`
- Added method `recvSpecLoadUpdate()`
- Added method `RubyPort:ClockedObject:makeSpecLoadUpdate()`

### `gem5/src/mem/ruby/system/Sequencer.cc`
- Modified the `issueRequest()` method to check for speculative loads and store the existing `L1CacheEntry` pointer for the desired address. 
- Added method `makeSpecLoadUpdate()` which checks for the speculative state of the incoming `Packet` and either restores or commits the load.

### `gem5/src/mem/ruby/system/Sequencer.hh`
- Added structure `SpeculativeRequest`:
```
struct SpeculativeRequest
{
	AbstractCacheEntry* l1CacheEntry;
	SpeculativeRequestStatus status;
};
```
- Added enum `SpeculativeRequestStatus: {Issued, Squashed, Committed}`.
- Added victim cache `std::map<Addr, SpeculativeRequest>` as class attribute.

### `gem5/src/mem/slicc/symbols/Func.py`
- Added the ability to set generated functions as public by absorbing the `public="yes"` flag from methods:
```
if self.public:
	return "public:\n%s %s(%s);\nprivate:\n" % (return_type, self.c_name,
		", ".join(self.param_strings))
else:
	return "%s %s(%s);" % (return_type, self.c_name,
```

## Evaluation

This code was tested with a set of 10 simple binaries of compiled C programs, since the implementation falls short in its ability to actually mitigate Spectre or Meltdown attacks, despite being a good set of steps in the correct direction. As such, the command used to observe working code changes was:
```
 ./build/X86/gem5.opt --debug-flags=RubySequencer ./configs/example/se.py --cmd=[BINARY_NAME] --cpu-type=O3CPU --caches --ruby
```

## Building
We used the standard Gem5 build process using scons. We built the simulator for X86 with the `j9` flag set.

## Output
During the running of a given binary, you may observe messages indicating that a specualtive load has been recieved at the Sequencer, and when loads are either quashed or commited when entries in the victim cache are restored or removed.
