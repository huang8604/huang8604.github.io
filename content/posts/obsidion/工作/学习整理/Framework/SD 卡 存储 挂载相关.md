---
tags:
  - blog
  - 案例分析
title: SD 卡 存储 挂载相关
date: 2025-01-02T09:33:48.826Z
lastmod: 2025-01-02T09:46:42.314Z
---
背景：

* **测试步骤:**

  1、插入sd卡\
  2、观察sd卡名字 （Samsung）\
  3、格式化sd卡

* **实测结果:**

  格式化SDcard后 名字变成Android

* **预期结果:**

  格式化SDcard后 名字不会改变

格式化SD后，sd卡显示的名称会变化。\
这个问题跟踪了一下格式化的流程：

```java
//SYSTEM/packages/apps/Settings/src/com/android/settings/deviceinfo/StorageWizardFormatConfirm.java

builder.setPositiveButton(
		TextUtils.expandTemplate(getText(R.string.storage_menu_format_option),
				disk.getShortDescription()),
		(dialog, which) -> {
			final Intent intent = new Intent(context, StorageWizardFormatProgress.class);
			intent.putExtra(EXTRA_DISK_ID, diskId);
			intent.putExtra(EXTRA_FORMAT_FORGET_UUID, formatForgetUuid);
			intent.putExtra(EXTRA_FORMAT_PRIVATE, formatPrivate);
			context.startActivity(intent);
		});

```

```java
///mnt/media/FH22/AP/SYSTEM/packages/apps/Settings/src/com/android/settings/deviceinfo/StorageWizardFormatProgress.java

setHeaderText(R.string.storage_wizard_format_progress_title, getDiskShortDescription());
setBodyText(R.string.storage_wizard_format_progress_body, getDiskDescription());
setBackButtonVisibility(View.INVISIBLE);
setNextButtonVisibility(View.INVISIBLE);
mTask = (PartitionTask) getLastCustomNonConfigurationInstance();
if (mTask == null) {
	mTask = new PartitionTask();
	mTask.setActivity(this);
	mTask.execute();
} else {
	mTask.setActivity(this);
}

@Override
protected Exception doInBackground(Void... params) {
	final StorageWizardFormatProgress activity = mActivity;
	final StorageManager storage = mActivity.mStorage;
	try {
		if (activity.mFormatPrivate) {
			storage.partitionPrivate(activity.mDisk.getId());
			publishProgress(40);

			final VolumeInfo privateVol = activity.findFirstVolume(TYPE_PRIVATE, 50);
			final CompletableFuture<PersistableBundle> result = new CompletableFuture<>();
			if(null != privateVol) {
				storage.benchmark(privateVol.getId(), new IVoldTaskListener.Stub() {
					@Override
					public void onStatus(int status, PersistableBundle extras) {
						// Map benchmark 0-100% progress onto 40-80%
						publishProgress(40 + ((status * 40) / 100));
					}

					@Override
					public void onFinished(int status, PersistableBundle extras) {
						result.complete(extras);
					}
				});
				mPrivateBench = result.get(60, TimeUnit.SECONDS).getLong("run",
						Long.MAX_VALUE);
			}

			// If we just adopted the device that had been providing
			// physical storage, then automatically move storage to the
			// new emulated volume.
			if (activity.mDisk.isDefaultPrimary()
					&& Objects.equals(storage.getPrimaryStorageUuid(),
							StorageManager.UUID_PRIMARY_PHYSICAL)) {
				Log.d(TAG, "Just formatted primary physical; silently moving "
						+ "storage to new emulated volume");
				storage.setPrimaryStorageUuid(privateVol.getFsUuid(), new SilentObserver());
			}

		} else {
			storage.partitionPublic(activity.mDisk.getId());
		}
		return null;
	} catch (Exception e) {
		return e;
	}
}

```

> storage.partitionPublic(activity.mDisk.getId());

```java
///mnt/media/FH22/AP/SYSTEM/frameworks/base/core/java/android/os/storage/StorageManager.java

public void partitionPublic(String diskId) {
	try {
		mStorageManager.partitionPublic(diskId);
	} catch (RemoteException e) {
		throw e.rethrowFromSystemServer();
	}
}

///mnt/media/FH22/AP/SYSTEM/frameworks/base/services/core/java/com/android/server/StorageManagerService.java
public void partitionPublic(String diskId) {

	super.partitionPublic_enforcePermission();

	final CountDownLatch latch = findOrCreateDiskScanLatch(diskId);
	try {
		mVold.partition(diskId, IVold.PARTITION_TYPE_PUBLIC, -1);
		waitForLatch(latch, "partitionPublic", 3 * DateUtils.MINUTE_IN_MILLIS);
	} catch (Exception e) {
		Slog.wtf(TAG, e);
	}
}

```

我们根据sdka 的exfat格式，有个aidl 调用，然后会调用到exfat 的vold里面来出来format

```cpp
///mnt/media/FH22/AP/SYSTEM/system/vold/fs/Exfat.cpp

static const char* kMkfsPath = "/system/bin/mkfs.exfat";
static const char* kFsckPath = "/system/bin/fsck.exfat";

status_t Format(const std::string& source) {
    std::vector<std::string> cmd;
    cmd.push_back(kMkfsPath);
    cmd.push_back("-n");
    cmd.push_back("android");
    cmd.push_back(source);

    int rc = ForkExecvp(cmd);
    if (rc == 0) {
        LOG(INFO) << "Format OK";
        return 0;
    } else {
        LOG(ERROR) << "Format failed (code " << rc << ")";
        errno = EIO;
        return -1;
    }
    return 0;
}

```

```
cmd.push_back("-n");
cmd.push_back("android");
```

通过"/system/bin/mkfs.exfat 进行了格式化， 通过 -n 添加了标签

至于显示部分

```java
///mnt/media/FH22/AP/SYSTEM/frameworks/base/core/java/android/os/storage/StorageManager.java
    @UnsupportedAppUsage
    public @Nullable String getBestVolumeDescription(VolumeInfo vol) {
        if (vol == null) return null;
        //wentao 
        Log.d("wentao", "getBestVolumeDescription  vol :"+ vol.toString());

        // Nickname always takes precedence when defined
        if (!TextUtils.isEmpty(vol.fsUuid)) {
            final VolumeRecord rec = findRecordByUuid(vol.fsUuid);
            if (rec != null && !TextUtils.isEmpty(rec.nickname)) {
                Log.d("wentao", "getBestVolumeDescription  rec.nickname :"+ rec.nickname);
                return rec.nickname;
            }
        }

        if (!TextUtils.isEmpty(vol.getDescription())) {
            Log.d("wentao", "getBestVolumeDescription  vol.getDescription() :"+ vol.getDescription());
            return vol.getDescription();
        }

        if (vol.disk != null) {
            Log.d("wentao", "getBestVolumeDescription  vol.disk.getDescription() :"+ vol.disk.getDescription());
            return vol.disk.getDescription();
        }

        return null;
    }

```

getBestVolumeDescription 里面有个显示优先级：\
1 nickname  自定义的别名\
2 volume description  这个一般就是 volume的label\
3 disk description   这个一般就是特殊厂商有固定的厂商名称

```cpp
            // Our goal here is to give the user a meaningful label, ideally
            // matching whatever is silk-screened on the card.  To reduce
            // user confusion, this list doesn't contain white-label manfid.
            switch (manfid) {
                // clang-format off
                case 0x000003: mLabel = "SanDisk"; break;
                case 0x00001b: mLabel = "Samsung"; break;
                case 0x000028: mLabel = "Lexar"; break;
                case 0x000074: mLabel = "Transcend"; break;
                    // clang-format on
```

如果什么都没有，就会显示unknow。

android 在格式化后，会打上android 标签。所以才会以android显示
