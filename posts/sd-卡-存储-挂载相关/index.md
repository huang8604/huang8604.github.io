# SD 卡 存储 挂载相关

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
		(dialog, which) -&gt; {
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
			final CompletableFuture&lt;PersistableBundle&gt; result = new CompletableFuture&lt;&gt;();
			if(null != privateVol) {
				storage.benchmark(privateVol.getId(), new IVoldTaskListener.Stub() {
					@Override
					public void onStatus(int status, PersistableBundle extras) {
						// Map benchmark 0-100% progress onto 40-80%
						publishProgress(40 &#43; ((status * 40) / 100));
					}

					@Override
					public void onFinished(int status, PersistableBundle extras) {
						result.complete(extras);
					}
				});
				mPrivateBench = result.get(60, TimeUnit.SECONDS).getLong(&#34;run&#34;,
						Long.MAX_VALUE);
			}

			// If we just adopted the device that had been providing
			// physical storage, then automatically move storage to the
			// new emulated volume.
			if (activity.mDisk.isDefaultPrimary()
					&amp;&amp; Objects.equals(storage.getPrimaryStorageUuid(),
							StorageManager.UUID_PRIMARY_PHYSICAL)) {
				Log.d(TAG, &#34;Just formatted primary physical; silently moving &#34;
						&#43; &#34;storage to new emulated volume&#34;);
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

&gt; storage.partitionPublic(activity.mDisk.getId());

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
		waitForLatch(latch, &#34;partitionPublic&#34;, 3 * DateUtils.MINUTE_IN_MILLIS);
	} catch (Exception e) {
		Slog.wtf(TAG, e);
	}
}

```

我们根据sdka 的exfat格式，有个aidl 调用，然后会调用到exfat 的vold里面来出来format

```cpp
///mnt/media/FH22/AP/SYSTEM/system/vold/fs/Exfat.cpp

static const char* kMkfsPath = &#34;/system/bin/mkfs.exfat&#34;;
static const char* kFsckPath = &#34;/system/bin/fsck.exfat&#34;;

status_t Format(const std::string&amp; source) {
    std::vector&lt;std::string&gt; cmd;
    cmd.push_back(kMkfsPath);
    cmd.push_back(&#34;-n&#34;);
    cmd.push_back(&#34;android&#34;);
    cmd.push_back(source);

    int rc = ForkExecvp(cmd);
    if (rc == 0) {
        LOG(INFO) &lt;&lt; &#34;Format OK&#34;;
        return 0;
    } else {
        LOG(ERROR) &lt;&lt; &#34;Format failed (code &#34; &lt;&lt; rc &lt;&lt; &#34;)&#34;;
        errno = EIO;
        return -1;
    }
    return 0;
}

```

```
cmd.push_back(&#34;-n&#34;);
cmd.push_back(&#34;android&#34;);
```

通过&#34;/system/bin/mkfs.exfat 进行了格式化， 通过 -n 添加了标签

至于显示部分

```java
///mnt/media/FH22/AP/SYSTEM/frameworks/base/core/java/android/os/storage/StorageManager.java
    @UnsupportedAppUsage
    public @Nullable String getBestVolumeDescription(VolumeInfo vol) {
        if (vol == null) return null;
        //wentao 
        Log.d(&#34;wentao&#34;, &#34;getBestVolumeDescription  vol :&#34;&#43; vol.toString());

        // Nickname always takes precedence when defined
        if (!TextUtils.isEmpty(vol.fsUuid)) {
            final VolumeRecord rec = findRecordByUuid(vol.fsUuid);
            if (rec != null &amp;&amp; !TextUtils.isEmpty(rec.nickname)) {
                Log.d(&#34;wentao&#34;, &#34;getBestVolumeDescription  rec.nickname :&#34;&#43; rec.nickname);
                return rec.nickname;
            }
        }

        if (!TextUtils.isEmpty(vol.getDescription())) {
            Log.d(&#34;wentao&#34;, &#34;getBestVolumeDescription  vol.getDescription() :&#34;&#43; vol.getDescription());
            return vol.getDescription();
        }

        if (vol.disk != null) {
            Log.d(&#34;wentao&#34;, &#34;getBestVolumeDescription  vol.disk.getDescription() :&#34;&#43; vol.disk.getDescription());
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
            // user confusion, this list doesn&#39;t contain white-label manfid.
            switch (manfid) {
                // clang-format off
                case 0x000003: mLabel = &#34;SanDisk&#34;; break;
                case 0x00001b: mLabel = &#34;Samsung&#34;; break;
                case 0x000028: mLabel = &#34;Lexar&#34;; break;
                case 0x000074: mLabel = &#34;Transcend&#34;; break;
                    // clang-format on
```

如果什么都没有，就会显示unknow。

android 在格式化后，会打上android 标签。所以才会以android显示


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/sd-%E5%8D%A1-%E5%AD%98%E5%82%A8-%E6%8C%82%E8%BD%BD%E7%9B%B8%E5%85%B3/  

