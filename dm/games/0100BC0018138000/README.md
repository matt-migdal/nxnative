# Super Mario RPG

Title ID `0100BC0018138000`, update 1.0.1 (`v65536`), Unity 2021.3.15f1 IL2CPP, Nintendo SDK 14.3.1, NVN.

## Current status

On the NX2Apple Mac host with existing shader libraries, tested the first Bowser battle, sword cinematic, Mario’s house, world map, first return to Bowser’s Keep, Exor dialogue and bridge destruction, then back to the world map. The final Keep-return comparison uses five SDK cuts and a 1.5 GB application pool, with no address-region failures or crashes logged and peak live reservations of 3.47 GB. Current nxnative runtime validation and longer sessions remain open. The iPhone ran through the opening battle with the earlier three-cut/2 GB configuration but stalled on returning to the Keep; the final configuration still needs device validation. This remains a draft memory bring-up profile, not a full playability claim.

Two NVN functions report unimplemented: `nvnCommandBufferSetSubpixelPrecisionBias` and `nvnCommandBufferSetConservativeRasterDilate`.

## Memory and startup

The profile uses a 5 GB reservation, 4.375 GB alias region, 256 MB stack and NPDM’s 16 MB system resource. `nninitStartup` at `main+0x62de90` initializes StandardAllocator over `pool - used`.

The provisional hardware pool (`0xcd500000`) reserves `0xc9400000` bytes. Before the SDK cuts, `CentralHeap::Init` returns ENOMEM (12) after virtual reservation fails with `0x1803`; the uninstrumented host exits 42 before `nnMain`.

SDK build `2587a39ac2547a780ec0e4b77e22afab` has its AddressRegionManager constructor at `sdk+0x12d8a0`. Cuts at `+0x12d8b0` and `+0x12d94c` reduce the node-area limit and occupied length from `0x3fe00000` to `0x3e00000`. Three alignment cuts at `+0x12daa4`, `+0x12daac`, and `+0x12dab0` align all regions from 4 MB up to 2 MB, matching SDK 20’s large-region alignment. Each cut asserts the original instruction and is gated by the SDK build ID.

Reducing only the largest alignment class to 32 MB and reporting a 2 GB pool fixes initial asset loading (`main+0x627a74`, a null 32 MB allocation) but does not survive early scene transitions. A later null 16 MB allocation faults at `main+0x219b60` after the sword cinematic. Allocation/free probes show Unity’s N MB + 252 KB reservations wasting space under the older size classes: 179 live regions reserve 3.09 GB but end at alias +4.22 GB.

With 2 MB alignment and the 2 GB pool, leaving Mario’s house still fails: 253 live reservations total 3.879 GB and reach alias +4.37475 GB. `sdk+0x12db68` reports no fitting region for `0x103f000` bytes, followed by a bad buffer address at `h_nvnBufferGetAddress+0x4`. This is virtual address exhaustion; a running frame overlay does not establish that scene loading completed.

The final profile reports a 1.5 GB pool (`app_pool 0x60000000`), reducing StandardAllocator’s initial reservation to `0x5c000000` and leaving another 512 MB for Unity’s independent regions. The comparison loaded the world map, Keep exterior and bridge sequence, then returned to the map with no allocation-failure probes or crash markers. Existing Mac apps can apply the pool setting through `Resources/dm/env.txt` (`DM_PROF_APP_POOL`); rebuilt hosts take it from the profile. The current upstream host’s automatic StandardAllocator cap is above the original request in this layout and does not provide this split by itself.

## Found on this game

- [The SDK’s address-region node area](../../../docs/reference/memory.md#the-sdks-address-region-node-area): newly registered SDK build.
- [Sizing the pool and alias region for virtual address memory](../../../docs/reference/memory.md#sizing-the-pool-and-alias-region-for-virtual-address-memory): the 1.5 GB pool leaves room for scene transitions.
- [Size-class alignment doubles Unity’s regions](../../../docs/reference/memory.md#size-class-alignment-doubles-unitys-regions): Unity needs the SDK 20 alignment policy as well as the smaller StandardAllocator reservation.

## Open validation

Test the final configuration on iPhone and current nxnative, then verify longer sessions. Existing shader libraries were reused; no shader compilation was needed for the memory fix. The minimal upstream profile does not yet include the complete local shader input tables.
