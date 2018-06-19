## Making Fake Files appear
---------
### Useful resources:

You probably don't need this

**WinFSP Tutorial:** https://github.com/billziss-gh/winfsp/wiki/WinFsp-Tutorial
**WinFSP API:** http://www.secfs.net/winfsp/apiref/
**WIN DAAS Reference:** https://msdn.microsoft.com/en-us/library/windows/desktop/ee663264(v=vs.85).aspx

------------
### DIY

If you want to relive the experience you can go ahead and solve the task yourself, the goal is to make it so that fake files and directories appear in the new mounted drive.

First of all, you have to install WINFSP and test it out, check out the tutorial link above if you haven't done it yet.

Next thing up is editing passthrough.c in your winfsp/samples directory, open it up and start suffering.

If you want to go hard mode, just dive in there and figure it out, otherwise I'll list a progressively more hinty list of hints

<details>
 <summary>Hint 1:
</summary>
 
> You want to find the bit where WINFSP reads the directory, which in this case is the ReadDirectory() function.
</details>
<details>
 <summary>Hint 2:
</summary>
 
> There's a while loop soon after the ```  FindHandle = FindFirstFileW(FullPath, &FindData); ``` chunk of code, inside that loop is where you want to do your editing, for this loop iterates over files and pushes them into a buffer that cool stuff happens with later</details>
<details>
 <summary>Hint 3:
</summary>
 
```c
memset(DirInfo, 0, sizeof *DirInfo);
                Length = (ULONG)wcslen(FindData.cFileName);
                DirInfo->Size = (UINT16)(FIELD_OFFSET(FSP_FSCTL_DIR_INFO, FileNameBuf) + Length * sizeof(WCHAR)); // This measures the size of FileNameBuf, if you're having trouble remember that you have to use wide-type characters here so a string literal would be written as L"texthere"
                DirInfo->FileInfo.FileAttributes = FindData.dwFileAttributes; //File attributes, don't worry about this
                DirInfo->FileInfo.ReparseTag = 0;
                DirInfo->FileInfo.FileSize =
                    ((UINT64)FindData.nFileSizeHigh << 32) | (UINT64)FindData.nFileSizeLow;
                DirInfo->FileInfo.AllocationSize = (DirInfo->FileInfo.FileSize + ALLOCATION_UNIT - 1)
                    / ALLOCATION_UNIT * ALLOCATION_UNIT; //Don't worry about this, it has to do with memory allocation
                DirInfo->FileInfo.CreationTime = ((PLARGE_INTEGER)&FindData.ftCreationTime)->QuadPart; // Creation time, check the references for filetype stuff
                DirInfo->FileInfo.LastAccessTime = ((PLARGE_INTEGER)&FindData.ftLastAccessTime)->QuadPart; 
                DirInfo->FileInfo.LastWriteTime = ((PLARGE_INTEGER)&FindData.ftLastWriteTime)->QuadPart;
                DirInfo->FileInfo.ChangeTime = DirInfo->FileInfo.LastWriteTime;
                DirInfo->FileInfo.IndexNumber = 0;
                DirInfo->FileInfo.HardLinks = 0; //Remember this from OS?
                memcpy(DirInfo->FileNameBuf, FindData.cFileName, Length * sizeof(WCHAR)); // This is where the filename is buddy, change naimely FinData.cFileName is the filename and it is being copied into the buffer.

                if (!FspFileSystemFillDirectoryBuffer(&FileContext->DirBuffer, DirInfo, &DirBufferResult)) // This is the big boi, this function copies everything we just did into the proper buffer for the filesystem, think of it almost like a pushback into a vector.
                    break;
```
</details>
<details>
 <summary>Hint 4:
</summary>
 
> Look at the code here in this repository, you go down to the ReadDirectory() function, you'll be able to make sense of the changes now
</details>