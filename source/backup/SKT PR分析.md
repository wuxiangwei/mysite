[TOC]

# SKT PR分析

## feature

在ceph_features.h文件新增CEPH_FEATURE_QOS_DMC feature。

```c++
#define CEPH_FEATURES_ALL // 包含Ceph所有的Feature
```

```
SimplePolicyMessenger(Messenger)  // AysncMessenger为SimplePolicyMessenger的派生类
    |-- policy_map: map<int, Policy>  // entity_name_t::type -> Policy
        |--Policy
            |-- features_supported: uint64_t // 支持的ceph featues
```

