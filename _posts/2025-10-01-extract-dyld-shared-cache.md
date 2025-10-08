---
layout: post
title: "Extract dyld shared cache"
date: 2025-10-01 10:00:00 +0800
categories: learning
---

From macOS 11, Apple ships a generated cache of all built in dynamic libraries and excludes the originals.

# extract tool
> /usr/lib/dsc_extractor.bundle

# extract function
```c
void (*extract)(const char *cache_path, const char *output_path, void(^block)(int completed, int total));
```
# extract code
```c
#include <dlfcn.h>

void (*extract)(const char *cache_path, const char *output_path, void(^block)(int completed, int total));

int main(int argc, const char * argv[]) {
    const char *cache_path = "/System/Volumes/Preboot/Cryptexes/OS/System/Library/dyld/dyld_shared_cache_arm64e";
    const char *output_path = "/tmp/libraries";
    const char *extractor_path = "/usr/lib/dsc_extractor.bundle";
    void *handle = dlopen(extractor_path, RTLD_LAZY);
    extract = dlsym(handle, "dyld_shared_cache_extract_dylibs_progress");
    extract(cache_path, output_path, ^void(int completed, int total) {
        printf("extracted %d/%d\n", completed, total);
    });
    return 0;
}
```
# References
<https://github.com/keith/dyld-shared-cache-extractor>
