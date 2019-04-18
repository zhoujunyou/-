### RxPermission

栗子

```java
 new RxPermissions(this)
                .request(Manifest.permission.READ_PHONE_STATE,
                        Manifest.permission.WRITE_EXTERNAL_STORAGE,
                        Manifest.permission.ACCESS_FINE_LOCATION,
                        Manifest.permission.ACCESS_COARSE_LOCATION
                )
                .subscribe(granted -> {
                    if (granted) {
                      
                }, throwable -> {

                });
```



   ```java
   public RxPermissions(@NonNull Activity activity) {
           mRxPermissionsFragment = getRxPermissionsFragment(activity);
       }
   
       private RxPermissionsFragment getRxPermissionsFragment(Activity activity) {
           RxPermissionsFragment rxPermissionsFragment = findRxPermissionsFragment(activity);
           boolean isNewInstance = rxPermissionsFragment == null;
           if (isNewInstance) {
               rxPermissionsFragment = new RxPermissionsFragment();
               FragmentManager fragmentManager = activity.getFragmentManager();
               fragmentManager
                       .beginTransaction()
                       .add(rxPermissionsFragment, TAG)
                       .commitAllowingStateLoss();
               fragmentManager.executePendingTransactions();
           }
           return rxPermissionsFragment;
       }.
   ```
```java
   public Observable<Boolean> request(final String... permissions) {
        return Observable.just(TRIGGER).compose(ensure(permissions));
    }
```

```java
public <T> ObservableTransformer<T, Boolean> ensure(final String... permissions) {
        return new ObservableTransformer<T, Boolean>() {
            @Override
            public ObservableSource<Boolean> apply(Observable<T> o) {
                return request(o, permissions)
                        // Transform Observable<Permission> to Observable<Boolean>
                        .buffer(permissions.length)
                        .flatMap(new Function<List<Permission>, ObservableSource<Boolean>>() {
                            @Override
                            public ObservableSource<Boolean> apply(List<Permission> permissions) throws Exception {
                                if (permissions.isEmpty()) {
                                    // Occurs during orientation change, when the subject receives onComplete.
                                    // In that case we don't want to propagate that empty list to the
                                    // subscriber, only the onComplete.
                                    return Observable.empty();
                                }
                                // Return true if all permissions are granted.
                                for (Permission p : permissions) {
                                    if (!p.granted) {
                                        return Observable.just(false);
                                    }
                                }
                                return Observable.just(true);
                            }
                        });
            }
        };
    }
```

​	

```java
private Observable<Permission> request(final Observable<?> trigger, final String... permissions) {
        if (permissions == null || permissions.length == 0) {
            throw new IllegalArgumentException("RxPermissions.request/requestEach requires at least one input permission");
        }
        return oneOf(trigger, pending(permissions))
                .flatMap(new Function<Object, Observable<Permission>>() {
                    @Override
                    public Observable<Permission> apply(Object o) throws Exception {
                        return requestImplementation(permissions);
                    }
                });
    }
```

1. ```java
    private Observable<?> pending(final String... permissions) {
           for (String p : permissions) {
               if (!mRxPermissionsFragment.containsByPermission(p)) {
                   return Observable.empty();
               }
           }
           return Observable.just(TRIGGER);
       }
       
       private Observable<?> oneOf(Observable<?> trigger, Observable<?> pending) {
           if (trigger == null) {
               return Observable.just(TRIGGER);
           }
           return Observable.merge(trigger, pending);
       }
    ```
   ```

2. ```java
   @TargetApi(Build.VERSION_CODES.M)
       private Observable<Permission> requestImplementation(final String... permissions) {
           List<Observable<Permission>> list = new ArrayList<>(permissions.length);
           List<String> unrequestedPermissions = new ArrayList<>();
   
           // In case of multiple permissions, we create an Observable for each of them.
           // At the end, the observables are combined to have a unique response.
           for (String permission : permissions) {
               mRxPermissionsFragment.log("Requesting permission " + permission);
               if (isGranted(permission)) {
                   // Already granted, or not Android M
                   // Return a granted Permission object.
                   list.add(Observable.just(new Permission(permission, true, false)));
                   continue;
               }
   
               if (isRevoked(permission)) {
                   // Revoked by a policy, return a denied Permission object.
                   list.add(Observable.just(new Permission(permission, false, false)));
                   continue;
               }
   
               PublishSubject<Permission> subject = mRxPermissionsFragment.getSubjectByPermission(permission);
               // Create a new subject if not exists
               if (subject == null) {
                   unrequestedPermissions.add(permission);
                   subject = PublishSubject.create();
                   mRxPermissionsFragment.setSubjectForPermission(permission, subject);
               }
   
               list.add(subject);
           }
   
           if (!unrequestedPermissions.isEmpty()) {
               String[] unrequestedPermissionsArray = unrequestedPermissions.toArray(new String[unrequestedPermissions.size()]);
               requestPermissionsFromFragment(unrequestedPermissionsArray);
           }
           return Observable.concat(Observable.fromIterable(list));
       }
   ```

   ```java
     void requestPermissionsFromFragment(String[] permissions) {
           mRxPermissionsFragment.log("requestPermissionsFromFragment " + TextUtils.join(", ", permissions));
           mRxPermissionsFragment.requestPermissions(permissions);
       }
   ```
   ```java
   @TargetApi(Build.VERSION_CODES.M)
    public void onRequestPermissionsResult(int requestCode, @NonNull String permissions[], @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);

        if (requestCode != PERMISSIONS_REQUEST_CODE) return;

        boolean[] shouldShowRequestPermissionRationale = new boolean[permissions.length];

        for (int i = 0; i < permissions.length; i++) {
            shouldShowRequestPermissionRationale[i] = shouldShowRequestPermissionRationale(permissions[i]);
        }

        onRequestPermissionsResult(permissions, grantResults, shouldShowRequestPermissionRationale);
    }

    void onRequestPermissionsResult(String permissions[], int[] grantResults, boolean[] shouldShowRequestPermissionRationale) {
        for (int i = 0, size = permissions.length; i < size; i++) {
            log("onRequestPermissionsResult  " + permissions[i]);
            // Find the corresponding subject
            PublishSubject<Permission> subject = mSubjects.get(permissions[i]);
            if (subject == null) {
                // No subject found
                Log.e(RxPermissions.TAG, "RxPermissions.onRequestPermissionsResult invoked but didn't find the corresponding permission request.");
                return;
            }
            mSubjects.remove(permissions[i]);
            boolean granted = grantResults[i] == PackageManager.PERMISSION_GRANTED;
            subject.onNext(new Permission(permissions[i], granted, shouldShowRequestPermissionRationale[i]));
            subject.onComplete();
        }
    }
   ```
1. 创建1个没有视图的RxPermissionsFragment 用于动态申请权限

2. oneOf 和pending方法还不是很理解。merge操作符会逐个发射source的数据 如果pending返回的source返回不为空 也就是mRxPermissionsFragment.containsByPermission(p)为true  则下游的流程都会走两次

3. requestImplementation 如果权限已被授权或拒绝 会生成相应的Observable<Permission>  加到list中

   否则得到一个subject加入list中 并加到unrequestedPermissions中 再用mRxPermissionsFragment去申请权限。 走的api方法

4. 权限结果返回后走到onRequestPermissionsResult方法 得到每个permissions 相应的subject 并发射结果

5. 回到requestImplementation的Observable.concat(Observable.fromIterable(list))中  concat会在list中的源发射数据时按个发出结果。Observable.just的源直接发射到下游。subject会在被调用onNext时往下游发射。

6.  回到ensure 的buffer(permissions.length) 当上游数据发射个数达到指定长度时往下游发射列表。

7. 最后flatmap里的逻辑对所有结果进行整理 权限都通过返回true 否则false。

