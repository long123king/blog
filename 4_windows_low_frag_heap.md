# Windows Low Fragmentation Heap
## core data structure

        bp ntdll!RtlFreeHeap

It's 3 parameters are : 

        ; BOOLEAN RtlFreeHeap(
        ;   _In_     PVOID HeapHandle, --> rcx
        ;   _In_opt_ ULONG Flags,      --> rdx
        ;   _In_     PVOID HeapBase    --> r8
        ; );

A case in windbg:

        kd> r
        rax=00000000009168f0 rbx=0000000000000002 rcx=0000000000a60000
        rdx=0000000000000002 rsi=0000000000a60000 rdi=0000000000000050
        rip=0000000077903200 rsp=0000000001fde558 rbp=0000000000000000
         r8=000000000092bf50  r9=0000000000000000 r10=0000000000000000
        r11=00000000009168f0 r12=00000000000000a0 r13=00000000009168f0
        r14=000000000092bf50 r15=0000000000000000
        iopl=0         nv up ei pl zr na po nc
        cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
        ntdll!RtlFreeHeap:
        0033:00000000`77903200 488bc4          mov     rax,rsp
        kd> dt _HEAP %rcx
        ntdll!_HEAP
           +0x000 Entry            : _HEAP_ENTRY
           +0x010 SegmentSignature : 0xffeeffee
           +0x014 SegmentFlags     : 0
           +0x018 SegmentListEntry : _LIST_ENTRY [ 0x00000000`008f0018 - 0x00000000`00a60128 ]
           +0x028 Heap             : 0x00000000`00a60000 _HEAP
           +0x030 BaseAddress      : 0x00000000`00a60000 Void
           +0x038 NumberOfPages    : 0x10
           +0x040 FirstEntry       : 0x00000000`00a60a80 _HEAP_ENTRY
           +0x048 LastValidEntry   : 0x00000000`00a70000 _HEAP_ENTRY
           +0x050 NumberOfUnCommittedPages : 0
           +0x054 NumberOfUnCommittedRanges : 1
           +0x058 SegmentAllocatorBackTraceIndex : 0
           +0x05a Reserved         : 0
           +0x060 UCRSegmentList   : _LIST_ENTRY [ 0x00000000`00a6ffe0 - 0x00000000`00a6ffe0 ]
           +0x070 Flags            : 0x1002
           +0x074 ForceFlags       : 0
           +0x078 CompatibilityFlags : 0
           +0x07c EncodeFlagMask   : 0x100000
           +0x080 Encoding         : _HEAP_ENTRY
           +0x090 PointerKey       : 0x198fd1d0`33ef8cfe
           +0x098 Interceptor      : 0
           +0x09c VirtualMemoryThreshold : 0xff00
           +0x0a0 Signature        : 0xeeffeeff
           +0x0a8 SegmentReserve   : 0x200000
           +0x0b0 SegmentCommit    : 0x2000
           +0x0b8 DeCommitFreeBlockThreshold : 0x400
           +0x0c0 DeCommitTotalFreeThreshold : 0x1000
           +0x0c8 TotalFreeSize    : 0x95f
           +0x0d0 MaximumAllocationSize : 0x000007ff`fffdefff
           +0x0d8 ProcessHeapsListIndex : 4
           +0x0da HeaderValidateLength : 0x208
           +0x0e0 HeaderValidateCopy : (null) 
           +0x0e8 NextAvailableTagIndex : 0
           +0x0ea MaximumTagIndex  : 0
           +0x0f0 TagEntries       : (null) 
           +0x0f8 UCRList          : _LIST_ENTRY [ 0x00000000`00958fd0 - 0x00000000`00958fd0 ]
           +0x108 AlignRound       : 0x1f
           +0x110 AlignMask        : 0xffffffff`fffffff0
           +0x118 VirtualAllocdBlocks : _LIST_ENTRY [ 0x00000000`00a60118 - 0x00000000`00a60118 ]
           +0x128 SegmentList      : _LIST_ENTRY [ 0x00000000`00a60018 - 0x00000000`008f0018 ]
           +0x138 AllocatorBackTraceIndex : 0
           +0x13c NonDedicatedListLength : 0
           +0x140 BlocksIndex      : 0x00000000`00a60230 Void
           +0x148 UCRIndex         : 0x00000000`00a60a90 Void
           +0x150 PseudoTagEntries : (null) 
           +0x158 FreeLists        : _LIST_ENTRY [ 0x00000000`0090e6d0 - 0x00000000`00950c40 ]
           +0x168 LockVariable     : 0x00000000`00a60208 _HEAP_LOCK
           +0x170 CommitRoutine    : 0x198fd1d0`33ef8cfe     long  +198fd1d033ef8cfe
           +0x178 FrontEndHeap     : 0x00000000`008f0080 Void
           +0x180 FrontHeapLockCount : 0
           +0x182 FrontEndHeapType : 0x2 ''
           +0x188 Counters         : _HEAP_COUNTERS
           +0x1f8 TuningParameters : _HEAP_TUNING_PARAMETERS

Set breakpoint at where the decoding happens:

        bp ntdll!RtlFreeHeap+0xa4

code is as below:

        kd> u ntdll!RtlFreeHeap+0xa4 L0a
        ntdll!RtlFreeHeap+0xa4:
        00000000`779032a4 488b4208        mov     rax,qword ptr [rdx+8]
        00000000`779032a8 488bda          mov     rbx,rdx
        00000000`779032ab 48b9ffffffffff000000 mov rcx,0FFFFFFFFFFh
        00000000`779032b5 4833de          xor     rbx,rsi
        00000000`779032b8 4823c1          and     rax,rcx
        00000000`779032bb 48c1eb04        shr     rbx,4
        00000000`779032bf 4833d8          xor     rbx,rax
        00000000`779032c2 48331dfff00d00  xor     rbx,qword ptr [ntdll!RtlpLFHKey (00000000`779e23c8)]
        00000000`779032c9 48c1e304        shl     rbx,4
        00000000`779032cd 0f0d0b          prefetchw [rbx]
        

context we got (exactly before ntdll!RtlFreeHeap+0xb8\[0033:00000000`779032b8\]):

        kd> r
        rax=900000f45073425e rbx=000000000092bf40 rcx=000000ffffffffff
        rdx=000000000092bf40 rsi=0000000000a60000 rdi=000000000092bf50
        rip=00000000779032b5 rsp=0000000001fde4e0 rbp=0000000000000002
         r8=000000000092bf50  r9=0000000000000000 r10=0000000000000000
        r11=00000000009168f0 r12=00000000000000a0 r13=00000000009168f0
        r14=000000000092bf50 r15=0000000000000000
        iopl=0         nv up ei ng nz ac po nc
        cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000296

source of some important registers:

        HeapHandle                  --> rsi
        (DWORD)_HEAP_ENTRY + 0x08   --> rax
        HeapBase - 0x10             --> rbx
        HeapBase                    --> rdi, r8

The LFH Key:

        kd> dq ntdll!RtlpLFHKey L1
        00000000`779e23c8  000000f4`507a615d

        kd> dt _HEAP_ENTRY r8-0x10
        ntdll!_HEAP_ENTRY
           +0x000 PreviousBlockPrivateData : (null) 
           +0x008 Size             : 0x425e
           +0x00a Flags            : 0x73 's'
           +0x00b SmallTagIndex    : 0x50 'P'
           +0x00c PreviousSize     : 0xf4
           +0x00e SegmentOffset    : 0 ''
           +0x00e LFHFlags         : 0 ''
           +0x00f UnusedBytes      : 0x90 ''
           +0x008 CompactHeader    : 0x900000f4`5073425e
           +0x000 Reserved         : (null) 
           +0x008 FunctionIndex    : 0x425e
           +0x00a ContextValue     : 0x5073
           +0x008 InterceptorValue : 0x5073425e
           +0x00c UnusedBytesLength : 0xf4
           +0x00e EntryOffset      : 0 ''
           +0x00f ExtendedBlockSignature : 0x90 ''
           +0x000 ReservedForAlignment : (null) 
           +0x008 Code1            : 0x5073425e
           +0x00c Code2            : 0xf4
           +0x00e Code3            : 0 ''
           +0x00f Code4            : 0x90 ''
           +0x008 AgregateCode     : 0x900000f4`5073425e


Translate this block to C++ code:

        typedef struct {
               BYTE _reserved_1[8];
               union
               {
                  struct
                  {
                     BYTE _reserved_2[6];
                     BYTE _segmentOffset;
                     BYTE _unusedBytes;     // 0x80: Free, 0x90: InUse
                  };
                  int64 _address;
               }
        } _HEAP_ENTRY;

        
         //(((&_HEAP ^ &_HEAP_ENTRY) >> 4) ^ (_HEAP_ENTRY._address & 0xFF FFFF FFFF) ^ (RtlpLFHKey)) << 4

        
        struct _HEAP_ENTRY* entry = (struct _HEAP_ENTRY*)(HeapBase - 0x10);


        void* temp1 = HeapHandle ^ HeapBase;
        void* temp2 = entry->_address & 0x0000 00FF FFFF FFFF;
        void* temp3 = temp1 / 16;
        void* temp4 = *((QWORD*)ntdll!RtlpLFHKey);


        struct _HEAP_SUBSEGMENT* sub_segment = (struct _HEAP_SUBSEGMENT*)((temp3 ^ temp2 ^ temp4) * 16);

        
Actually, \_HEAP\_USERDATA\_HEADER is just UserBlocks, which holds several \_HEAP\_ENTRYs, free or allocated.
        
        _HEAP_USERDATA_HEADER   --> 0x00000000`0092b620
        _HEAP_ENTRY             --> 0x00000000`0092bf40  

        ntdll!_HEAP_USERDATA_HEADER
           +0x000 SFreeListEntry   : _SINGLE_LIST_ENTRY
           +0x000 SubSegment       : 0x00000000`00a68f70 _HEAP_SUBSEGMENT
           +0x008 Reserved         : 0x00000000`0092ac10 Void
           +0x010 SizeIndex        : 0xc
           +0x018 Signature        : 0xf0e0d0c0

        kd> dt _HEAP_SUBSEGMENT 0x00000000`00a68f70
        ntdll!_HEAP_SUBSEGMENT
           +0x000 LocalInfo        : 0x00000000`008f0860 _HEAP_LOCAL_SEGMENT_INFO
           +0x008 UserBlocks       : 0x00000000`0092b620 _HEAP_USERDATA_HEADER
           +0x010 AggregateExchg   : _INTERLOCK_SEQ
           +0x018 BlockSize        : 6
           +0x01a Flags            : 0
           +0x01c BlockCount       : 0x2a
           +0x01e SizeIndex        : 0x5 ''
           +0x01f AffinityIndex    : 0 ''
           +0x018 Alignment        : [2] 6
           +0x020 SFreeListEntry   : _SINGLE_LIST_ENTRY
           +0x028 Lock             : 7

very weird situation, the SizeIndex is NOT the same, one is 0xc, the other is 0x5.


So we know all the chunks in this UserBlocks are of the same size:
            
            0x0C * 0x08 = 0x60 bytes

so every chunk is 0x60 bytes, including the \_HEAP\_ENTRY prefix.
and \_HEAP\_ENTRY.\_unusedBytes indicates if it is free or not.

We have already known \_HEAP, so we can get further look at \_LFH\_HEAP.

        kd> dt _HEAP 0000000000a60000
        ntdll!_HEAP
           +0x000 Entry            : _HEAP_ENTRY
           +0x010 SegmentSignature : 0xffeeffee
           +0x014 SegmentFlags     : 0
           +0x018 SegmentListEntry : _LIST_ENTRY [ 0x00000000`008f0018 - 0x00000000`00a60128 ]
           +0x028 Heap             : 0x00000000`00a60000 _HEAP
           +0x030 BaseAddress      : 0x00000000`00a60000 Void
           +0x038 NumberOfPages    : 0x10
           +0x040 FirstEntry       : 0x00000000`00a60a80 _HEAP_ENTRY
           +0x048 LastValidEntry   : 0x00000000`00a70000 _HEAP_ENTRY
           +0x050 NumberOfUnCommittedPages : 0
           +0x054 NumberOfUnCommittedRanges : 1
           +0x058 SegmentAllocatorBackTraceIndex : 0
           +0x05a Reserved         : 0
           +0x060 UCRSegmentList   : _LIST_ENTRY [ 0x00000000`00a6ffe0 - 0x00000000`00a6ffe0 ]
           +0x070 Flags            : 0x1002
           +0x074 ForceFlags       : 0
           +0x078 CompatibilityFlags : 0
           +0x07c EncodeFlagMask   : 0x100000
           +0x080 Encoding         : _HEAP_ENTRY
           +0x090 PointerKey       : 0x198fd1d0`33ef8cfe
           +0x098 Interceptor      : 0
           +0x09c VirtualMemoryThreshold : 0xff00
           +0x0a0 Signature        : 0xeeffeeff
           +0x0a8 SegmentReserve   : 0x200000
           +0x0b0 SegmentCommit    : 0x2000
           +0x0b8 DeCommitFreeBlockThreshold : 0x400
           +0x0c0 DeCommitTotalFreeThreshold : 0x1000
           +0x0c8 TotalFreeSize    : 0x95f
           +0x0d0 MaximumAllocationSize : 0x000007ff`fffdefff
           +0x0d8 ProcessHeapsListIndex : 4
           +0x0da HeaderValidateLength : 0x208
           +0x0e0 HeaderValidateCopy : (null) 
           +0x0e8 NextAvailableTagIndex : 0
           +0x0ea MaximumTagIndex  : 0
           +0x0f0 TagEntries       : (null) 
           +0x0f8 UCRList          : _LIST_ENTRY [ 0x00000000`00958fd0 - 0x00000000`00958fd0 ]
           +0x108 AlignRound       : 0x1f
           +0x110 AlignMask        : 0xffffffff`fffffff0
           +0x118 VirtualAllocdBlocks : _LIST_ENTRY [ 0x00000000`00a60118 - 0x00000000`00a60118 ]
           +0x128 SegmentList      : _LIST_ENTRY [ 0x00000000`00a60018 - 0x00000000`008f0018 ]
           +0x138 AllocatorBackTraceIndex : 0
           +0x13c NonDedicatedListLength : 0
           +0x140 BlocksIndex      : 0x00000000`00a60230 Void
           +0x148 UCRIndex         : 0x00000000`00a60a90 Void
           +0x150 PseudoTagEntries : (null) 
           +0x158 FreeLists        : _LIST_ENTRY [ 0x00000000`0090e6d0 - 0x00000000`00950c40 ]
           +0x168 LockVariable     : 0x00000000`00a60208 _HEAP_LOCK
           +0x170 CommitRoutine    : 0x198fd1d0`33ef8cfe     long  +198fd1d033ef8cfe
           +0x178 FrontEndHeap     : 0x00000000`008f0080 Void
           +0x180 FrontHeapLockCount : 0
           +0x182 FrontEndHeapType : 0x2 ''
           +0x188 Counters         : _HEAP_COUNTERS
           +0x1f8 TuningParameters : _HEAP_TUNING_PARAMETERS

and we get \_LFH\_HEAP

        kd> dt _LFH_HEAP 0x00000000`008f0080 
        ntdll!_LFH_HEAP
           +0x000 Lock             : _RTL_CRITICAL_SECTION
           +0x028 SubSegmentZones  : _LIST_ENTRY [ 0x00000000`00a68bf0 - 0x00000000`0092ee20 ]
           +0x038 ZoneBlockSize    : 0x30
           +0x040 Heap             : 0x00000000`00a60000 Void
           +0x048 SegmentChange    : 0
           +0x04c SegmentCreate    : 0x29
           +0x050 SegmentInsertInFree : 0
           +0x054 SegmentDelete    : 0xc
           +0x058 CacheAllocs      : 0x1e
           +0x05c CacheFrees       : 0
           +0x060 SizeInCache      : 0x7f0
           +0x068 RunInfo          : _HEAP_BUCKET_RUN_INFO
           +0x070 UserBlockCache   : [12] _USER_MEMORY_CACHE_ENTRY
           +0x1f0 Buckets          : [128] _HEAP_BUCKET
           +0x3f0 LocalData        : [1] _HEAP_LOCAL_DATA

the struct \_HEAP\_LOCAL\_DATA is embedded in the \_LFH\_HEAP structure.

        kd> dt _HEAP_LOCAL_DATA
        ntdll!_HEAP_LOCAL_DATA
           +0x000 DeletedSubSegments : _SLIST_HEADER
           +0x010 CrtZone          : Ptr64 _LFH_BLOCK_ZONE
           +0x018 LowFragHeap      : Ptr64 _LFH_HEAP
           +0x020 Sequence         : Uint4B
           +0x030 SegmentInfo      : [128] _HEAP_LOCAL_SEGMENT_INFO

we can get the 0x00 record via:

        kd> dt _HEAP_LOCAL_SEGMENT_INFO 0x00000000`008f0080 + 0x3f0 + 0x30 + 0x0 * 0x0c0
        ntdll!_HEAP_LOCAL_SEGMENT_INFO
           +0x000 Hint             : 0x00000000`00a68f40 _HEAP_SUBSEGMENT
           +0x008 ActiveSubsegment : 0x00000000`00a68f40 _HEAP_SUBSEGMENT
           +0x010 CachedItems      : [16] 0x00000000`00a68c70 _HEAP_SUBSEGMENT
           +0x090 SListHeader      : _SLIST_HEADER
           +0x0a0 Counters         : _HEAP_BUCKET_COUNTERS
           +0x0a8 LocalData        : 0x00000000`008f0470 _HEAP_LOCAL_DATA
           +0x0b0 LastOpSequence   : 0x17
           +0x0b4 BucketIndex      : 0
           +0x0b6 LastUsed         : 2

but we get an empty record for 0x0c:

        kd> dt _HEAP_LOCAL_SEGMENT_INFO 0x00000000`008f0080 + 0x3f0 + 0x30 + 0xc * 0x0c0
        ntdll!_HEAP_LOCAL_SEGMENT_INFO
           +0x000 Hint             : (null) 
           +0x008 ActiveSubsegment : (null) 
           +0x010 CachedItems      : [16] (null) 
           +0x090 SListHeader      : _SLIST_HEADER
           +0x0a0 Counters         : _HEAP_BUCKET_COUNTERS
           +0x0a8 LocalData        : (null) 
           +0x0b0 LastOpSequence   : 0
           +0x0b4 BucketIndex      : 0
           +0x0b6 LastUsed         : 0

so it is 0x05 actually:

        kd> dt _HEAP_LOCAL_SEGMENT_INFO 0x00000000`008f0080 + 0x3f0 + 0x30 + 0x5 * 0x0c0
        ntdll!_HEAP_LOCAL_SEGMENT_INFO
           +0x000 Hint             : 0x00000000`00a68f70 _HEAP_SUBSEGMENT
           +0x008 ActiveSubsegment : 0x00000000`00a68f70 _HEAP_SUBSEGMENT
           +0x010 CachedItems      : [16] 0x00000000`00a68cd0 _HEAP_SUBSEGMENT
           +0x090 SListHeader      : _SLIST_HEADER
           +0x0a0 Counters         : _HEAP_BUCKET_COUNTERS
           +0x0a8 LocalData        : 0x00000000`008f0470 _HEAP_LOCAL_DATA
           +0x0b0 LastOpSequence   : 0x18
           +0x0b4 BucketIndex      : 5
           +0x0b6 LastUsed         : 0

what about the bucket structure:

        kd> dt _HEAP_BUCKET 0x00000000`008f0080 + 0x1f0 + 0x04 * 0x5
        ntdll!_HEAP_BUCKET
           +0x000 BlockUnits       : 6
           +0x002 SizeIndex        : 0x5 ''
           +0x003 UseAffinity      : 0y0
           +0x003 DebugFlags       : 0y00

