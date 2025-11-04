---
title: "avb校验相关与块校验原理_fec: recursion too deep-CSDN博客"
source: https://blog.csdn.net/bme314/article/details/128745809
published: 
created: 2025-11-04
description: 
tags:
  - clippings
  - blog
  - 转载
date: 2025-11-04T03:27:12.202Z
lastmod: 2025-11-04T04:00:42.201Z
---
文章详细阐述了Linux系统在启动过程中针对块设备的校验流程，涉及到VerifiedBoot.c和LinuxLoader.c等关键组件。通用块设备层处理I/O请求，包括扇区、块、段和数据页的概念。动态校验流程中，verity\_end\_io函数用于处理错误并触发工作队列进行校验。dm-verity用于保证数据完整性，通过创建哈希树并配置目标表来验证块设备上的数据。此外，文章还提到了init用户态流程和清除panic标识的方法。

### 一、启动校验流程

```c
edk2/QcomModulePkg/Library/avb/VerifiedBoot.c
DEBUG ((EFI_D_ERROR, "LoadImageAndAuth failed %r\n", Status)); in LoadImageAndAuth()

edk2/QcomModulePkg/Application/LinuxLoader/LinuxLoader.c
DEBUG ((EFI_D_ERROR, "LoadImageAndAuth failed: %r\n", Status));

BootLib/PartitionTableUpdate.c
/QcomModulePkg/Library/avb/libavb/avb_slot_verify.c
LinuxLoaderEntry
    Status = GetRebootReason (&BootReason);
    校验启动原因
    EnumeratePartitions()
    FindPtnActiveSlot
    UpdatePartitionEntries()   -----   加载avb分区
    LoadImageAndAuth      ---------------                  VerifiedBoot.c 
        FindBootableSlot    --------------         PartitionTableUpdate.c//error
            GetBootPartitionEntry 
            HandleActiveSlotUnbootable          //一直返回错误
                GetBootPartitionEntry             // BootEntry       检测没过
                GetActiveSlot                     //success
                    GetBootPartitionEntry    -----    for()
        LoadImageAndAuthVB2  ---------------           VerifiedBoot.c 
            avb_should_update_rollback
                IsCurrentSlotBootable
            avb_slot_verify    ------------------      avb_slot_verify.c
                load_and_verify_vbmeta
                    ops->read_from_partition(ops,  
                        AvbReadFromPartition
                           "Load Image %a total time: %lu ms \n"
                    avb_vbmeta_image_verify
                        avb_rsa_verify
                            iavb_parse_key_data                 //Unexpected key length
                avb_manage_hashtree_error_mode
                avb_append_options   //avb 根据resolved_hashtree_error_mode 设置verity_mode
                    cmdline_append_option    //设置verity_mode到slot_data->cmdline
                avb_sub_cmdline
            HandleActiveSlotUnbootable                //当前槽置为unbootable
            AppendVBCmdLine      //cmdline赋值到VBCmdLine
        DisplayVerifiedBootScreen
            DisplayVerifiedBootMenu
    BootLinux (&Info);
         UpdateCmdLine
             AddtoBootConfigList   //gBS = SystemTable->BootServices;这里会利用uefi boot服务记录map
c12345678910111213141516171819202122232425262728293031323334353637383940414243
```

### 二、通用块设备层对请求的处理

#### 2.1 块设备概述

![5e00f2e29c3ca54ef8bcc80bf046188f\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a7e546.png)

![acc3989c58db48303c93909ac34381b5\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a5a748.png)\
• 块设备的 [控制器](https://so.csdn.net/so/search?q=%E6%8E%A7%E5%88%B6%E5%99%A8\&spm=1001.2101.3001.7020) 传输的固定数据单元大小称为扇区（sector）。因此I/O调度器\
和块 [设备驱动](https://so.csdn.net/so/search?q=%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8\&spm=1001.2101.3001.7020) 必须以扇区为单位管理数据。\
• [虚拟文件系统](https://so.csdn.net/so/search?q=%E8%99%9A%E6%8B%9F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F\&spm=1001.2101.3001.7020) 、映射层（mapping layer）管理磁盘数据的逻辑单元大小称为块\
（block）。对于文件系统来说，块是最小的磁盘数据存储单元。\
• 前面在分散/聚合DMA中，我们提到块设备驱动应该能够处理称为“段”的数据\
单元；每个“段”是内存中的一页或页的一部分，“段”中的数据在磁盘上是\
连续的。\
• 磁盘缓冲区处理的数据单元大小为“页”，每个对应一个页帧。（注1）\
• 通用块设备层粘合所有上层和底层的部分，这样它就知道扇区、块、段和数据\
页。

![954834b855425f1c294027430bc33a92\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a9b14c.png)\
bi\_io\_vecs指向一个bio\_vec 结构体 数组，该结构体链表包含了一个特定I/O操作所需要\
使用到的所有段（segment）。每个bio\_vec结构都是一个形式为\<page, offset, len>的向量，\
它描述的是一个特定的段：段所在的物理页、块在物理页中的偏移量、从给定偏移量开始的\
块长度。整个bio\_io\_vec结构体数组表示了一个完整的缓冲区。bio\_vec结构体定义在文件\
include/linux/bio.h中。

内核使用gendisk结构，定义在include/linux/genhd.h中，来表示一个独立的磁盘设备。\
实际上内核还使用gendisk表示分区，但是驱动程序不需要了解这些。在gendisk结构中的许\
多成员必须由驱动程序进行初始化。\
物理磁盘通常被分成多个逻辑分区。每个块设备文件可以表示一个整个物理磁盘或者其\
中的一个分区。如/dev/sda、/dev/sda1、/dev/sda2等。若一个物理磁盘有多个分区，则磁\
盘的布局保存在hd\_struct数据结构数组中，数组的地址由gendisk结构体中的part成员保存。\
hd\_struct数据结构的定义在文件include/linux/genhd.h中。\
![1fc10005749f6d7feb2679226952a87e\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a5898c.png)\
当内核在系统中发现一个新磁盘时，调用alloc\_disk（）分配相关数据结构，如gendisk，\
hd\_struct等，然后调用add\_disk（）将磁盘添加到系统中。注意：一旦调用了add\_disk，\
磁盘设备就被“激活“，表示可以使用，并随时会调用它们提供的方法。

#### 2.2向通用块设备层发送请求

1. 上层下发磁盘数据请求
2. 通用块层申请bio结构，将请求的数据分段记录到bio中
3. 如果请求的数据大于一个bio允许的最大数据量，则将请求分成多个bio
4. 调用submit\_bio提交bio请求
5. submit\_bio函数经过层层调用，最终调用块设备请求队列中的make\_request\_fn成员函数将bio提交给I/O调度层进行处理

generic\_make\_request函数将bio连接到current->bio\_list链表中，并调用\_\_generic\_make\_request函数提交链表中所有的bio。\
![42e5e295f7c8fdf163477337556cd667\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a53dd6.png)\
\_\_generic\_make\_request函数最终会调用块设备的请求队列中的make\_request\_fn成员函数将bio请求发送给I/O调度层，至此对磁盘的数据请求离开通用块层，进入下一层——I/O调度层\
![8eb9347c4d3a718659464cbfdb6eb4ff\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a9cc7b.png)

#### 2.3 动态校验流程

```c
verity_map
    bio->bi_end_io = verity_end_io;

verity_end_io
    INIT_WORK(&io->work, verity_work);
    queue_work(io->v->verify_wq, &io->work);

verity_work
    verity_finish_io
        verity_verify_io
            struct bio *bio = dm_bio_from_per_bio_data
            for (b = 0; b < io->n_blocks; b++)
                verity_fec_decode
                    DMWARN_LIMIT("%s: FEC: recursion too deep", v->data_dev->name);
                verity_handle_err
                    DMERR_LIMIT("%s: %s block %llu is corrupted", v->data_dev->name,
                                type_str, block);

__submit_bio
    blk_mq_submit_bio
        bio_endio                                        ///kernel_platform/msm-kernel/block/bio.c
            bio->bi_end_io
c12345678910111213141516171819202122
```

```c
struct dm_verity_io {
72      struct dm_verity *v;
73  
74      /* original value of bio->bi_end_io */
75      bio_end_io_t *orig_bi_end_io;
76  
77      sector_t block;
78      unsigned n_blocks;
79  
80      struct bvec_iter iter;
81  
82      struct work_struct work;
83  
84      /*
85       * Three variably-size fields follow this struct:
86       *
87       * u8 hash_req[v->ahash_reqsize];
88       * u8 real_digest[v->digest_size];
89       * u8 want_digest[v->digest_size];
90       *
91       * To access them use: verity_io_hash_req(), verity_io_real_digest()
92       * and verity_io_want_digest().
93       */
94  };
c123456789101112131415161718192021222324
```

```c
int verity_hash_for_block(struct dm_verity *v, struct dm_verity_io *io,
335                sector_t block, u8 *digest, bool *is_zero)
336  {
337      int r = 0, i;
339      if (likely(v->levels)) {
340          /*
341           * First, we try to get the requested hash for
342           * the current block. If the hash block itself is
343           * verified, zero is returned. If it isn't, this
344           * function returns 1 and we fall back to whole
345           * chain verification.
346           */
347          r = verity_verify_level(v, io, block, 0, true, digest);
348          if (likely(r <= 0))
349              goto out;
350      }
352      memcpy(digest, v->root_digest, v->digest_size);
354      for (i = v->levels - 1; i >= 0; i--) {
355          r = verity_verify_level(v, io, block, i, false, digest);
356          if (unlikely(r))
357              goto out;
358      }
359  out:
360      if (!r && v->zero_digest)
361          *is_zero = !memcmp(v->zero_digest, digest, v->digest_size);
362      else
363          *is_zero = false;
364  
365      return r;
366  }
c123456789101112131415161718192021222324252627282930
```

#### 2.4 核心校验数据

```c
struct dm_verity {
35      struct dm_dev *data_dev;
36      struct dm_dev *hash_dev;
37      struct dm_target *ti;
38      struct dm_bufio_client *bufio;
39      char *alg_name;
40      struct crypto_ahash *tfm;
41      u8 *root_digest;    /* digest of the root block */
42      u8 *salt;        /* salt: its size is salt_size */
43      u8 *zero_digest;    /* digest for a zero block */
44      unsigned salt_size;
45      sector_t data_start;    /* data offset in 512-byte sectors */
46      sector_t hash_start;    /* hash start in blocks */
47      sector_t data_blocks;    /* the number of data blocks */
48      sector_t hash_blocks;    /* the number of hash blocks */
49      unsigned char data_dev_block_bits;    /* log2(data blocksize) */
50      unsigned char hash_dev_block_bits;    /* log2(hash blocksize) */
51      unsigned char hash_per_block_bits;    /* log2(hashes in hash block) */
52      unsigned char levels;    /* the number of tree levels */
53      unsigned char version;
54      unsigned digest_size;    /* digest size for the current hash algorithm */
55      unsigned int ahash_reqsize;/* the size of temporary space for crypto */
56      int hash_failed;    /* set to 1 if hash of any block failed */
57      enum verity_mode mode;    /* mode for handling verification errors */
58      unsigned corrupted_errs;/* Number of errors for corrupted blocks */
59
60      struct workqueue_struct *verify_wq;
61
62      /* starting blocks for each tree level. 0 is the lowest level. */
63      sector_t hash_level_block[DM_VERITY_MAX_LEVELS];
64
65      struct dm_verity_fec *fec;    /* forward error correction */
66      unsigned long *validated_blocks; /* bitset blocks validated */
67
68      char *signature_key_desc; /* signature keyring reference */
69  };
70
c12345678910111213141516171819202122232425262728293031323334353637
```

```c
[??? 3.809367] v->hash_blocks = 662070?? （662070-656896=5174）
[??? 3.812900] v->data_dev_block_bits=12
[??? 3.816608] v->hash_dev_block_bits=12
[??? 3.820279] v->hash_per_block_bits=7?? (4096/32 = 128, 指一个4K块上有128个哈希值)
[??? 3.823888] v->levels=3
[??? 3.826380] v->version =1
[??? 3.829033] v->digest_size=32
c1234567
```

```c
数据块有656896X4X1024? (超过了2G)
Level 0 :? 656896/128 = 5132个4k块
Level 1 :? 5132/128= 41个4k块
Level 2:? 40 /128 = 0，即1个4k块
共 5132+41+1 = 5174 个块
c12345
```

```c
static void verity_hash_at_level(struct dm_verity *v, sector_t block, int level,
193                   sector_t *hash_block, unsigned *offset)
194  {
195      sector_t position = verity_position_at_level(v, block, level);
196      unsigned idx;
198      *hash_block = v->hash_level_block[level] + (position >> v->hash_per_block_bits);
200      if (!offset)
201          return;
203      idx = position & ((1 << v->hash_per_block_bits) - 1);
204      if (!v->version)
205          *offset = idx * v->digest_size;
206      else
207          *offset = idx << (v->hash_dev_block_bits - v->hash_per_block_bits);
208  }

static sector_t verity_position_at_level(struct dm_verity *v, sector_t block,
92                       int level)
93  {
94      return block >> (level * v->hash_per_block_bits);
95  }
c1234567891011121314151617181920
```

### 三、init用户态流程

```c
FirstStageMain
    FirstStageMount::DoFirstStageMount
        FirstStageMount::MountPartitions
            FirstStageMount::MountPartition
                FirstStageMountVBootV2::SetUpDmVerity
                    AvbHandle::LoadAndVerifyVbmeta
                        Failed to load offline vbmeta for
c1234567
```

### 四 清除panic标识

```c
adb shell
reboot "dm-verity enforcing"
reboot "dm-verity device corrupted"
c123
```

### 五 dm-verity建立过程

在Device Mapper （DM）中使用dm-verity时，需要配置一个哈希树（hash tree），以便验证块设备上的数据的完整性。这通常通过在dmsetup命令中的目标表（target table）参数中指定相关参数来完成。以下是如何为dm-mapper中的dm-verity构建一个哈希树配置表的一般步骤：

1. **创建源块设备** ：首先，你需要创建一个普通的块设备，通常是一个已经存在的块设备，例如硬盘分区或者一个镜像文件。
2. **选择哈希算法** ：确定要使用的哈希算法。dm-verity支持多种哈希算法，包括SHA-256、SHA-1、SHA-512等。你需要选择一个适合你的需求的哈希算法。
3. **生成哈希表** ：使用生成 哈希表 的工具，例如veritysetup工具，来为源块设备生成哈希表。这个哈希表包含了块设备中每个块的哈希值。通常，你可以运行以下命令来生成哈希表：

```c
veritysetup format /dev/source_device /dev/mapper/verity_device
c1
```

其中，/dev/source\_device 是源块设备，/dev/mapper/verity\_device 是dm-verity设备。这个命令将生成哈希表并将其存储在dm-verity设备上。

1. **配置dm-verity目标** ：使用dmsetup命令配置dm-verity目标。你需要提供哈希树的根哈希值、块大小、源块设备和dm-verity设备的名称。通常，配置表看起来像这样：

```c
dmsetup create verity_device --table '0 <block_size> verity <root_hash> /dev/source_device /dev/mapper/verity_device'
c1
```

其中，\<block\_size> 是块的大小，\<root\_hash> 是哈希树的根哈希值。

1. **使用dm-verity设备** ：一旦dm-verity设备被创建，你可以像使用任何其他块设备一样使用它。数据将在读取时被验证，以确保其完整性。

这里的关键是生成哈希表并配置dm-verity目标，以便验证数据的完整性。哈希表是dm-verity验证数据完整性所必需的，因为它包含了源数据块的哈希值，用于验证数据是否被篡改。配置表指定了如何使用哈希表以及其他相关参数。配置表的格式取决于dm-verity目标的实现和你的需求，但通常是一个包含所需参数的文本字符串。

ubuntu 上dm-mapper相关信息：

```c
astro@astrox:~/code/test$ sudo lvdisplay --maps /dev/mapper/ubuntu--vg-ubuntu--lv
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                fSgsCj-fCWR-JeiN-WWQL-xPdv-T1gZ-QO9f9M
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2023-05-09 09:07:38 +0000
  LV Status              available
  # open                 1
  LV Size                49.25 GiB
  Current LE             12608
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0

  --- Segments ---
  Logical extents 0 to 12607:
    Type                linear
    Physical volume     /dev/sda3
    Physical extents    0 to 12607

c123456789101112131415161718192021222324
```

参考：\
https://aliez22.github.io/posts/11537/\
https://www.shili8.cn/article/detail\_20000194464.html

![5e00f2e29c3ca54ef8bcc80bf046188f\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a7e546.png) ![acc3989c58db48303c93909ac34381b5\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a5a748.png) ![954834b855425f1c294027430bc33a92\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a9b14c.png) ![1fc10005749f6d7feb2679226952a87e\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a5898c.png) ![42e5e295f7c8fdf163477337556cd667\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a53dd6.png) ![8eb9347c4d3a718659464cbfdb6eb4ff\_MD5](https://picgo.myjojo.fun:666/i/2025/11/04/6909737a9cc7b.png)
