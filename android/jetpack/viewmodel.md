# ViewModel



> ViewModel 的存在，主要是为了解决 状态托管 和 页面通信 的问题。



> 对于轻量的状态，可以通过视图控制器基类的 saveInstanceState 机制，以序列化的方式完成存储和恢复。

> 对于重量级的状态，例如通过网络请求得到的 List，可以通过生命周期长于视图控制器的 ViewModel 持有，从而得以直接从 ViewModel 恢复，而不是以效率较低的序列化方式。



ViewModel的设计思想，怎么实现作用域可控的？&#x20;

ViewModel怎么实现销毁的





[`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 类旨在以注重生命周期的方式存储和管理界面相关的数据。[`ViewModel`](https://developer.android.google.cn/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 类让数据可在发生屏幕旋转等配置更改后继续留存。





\


![](<../../.gitbook/assets/image (61).png>)



```
    // ViewModel
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$lifecycle_version"
```



```
val model: MainViewModel by viewModels()

注意这里只能是val, 不能使用var
```



{% embed url="https://developer.android.google.cn/topic/libraries/architecture/viewmodel?hl=zh-cn#kotlin" %}



![](<../../.gitbook/assets/image (349).png>)



![](<../../.gitbook/assets/image (324).png>)



### 核心数据结构

#### ViewModel

ViewModel 是一个抽象类，使用者需要继承它



#### ViewModelStore



#### ViewModelStoreOwner

ViewModelStore的持有者，



```
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

## ViewModelProvider







