# AccountManagerService解读

## 背景

> 本文主要为了解读系统账户管理方式，使用方式请自行搜索关键字AccountManager 或 SyncAdapter 使用方式
>
> * AccountManager作为系统账户管理方式，可以让本地化的账户信息更加安全
> * 作为多账户管理使用
> * 可以按照一定时机拉起同步功能
> * 自动起活

## 实现方式

1. AccountManager 是使用Binder方式跨进程访问

## 方法

1. 添加账户

```java
@Override
    public boolean addAccountExplicitly(Account account, String password, Bundle extras) {
        Bundle.setDefusable(extras, true);
        final int callingUid = Binder.getCallingUid();
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "addAccountExplicitly: " + account
                    + ", caller's uid " + callingUid
                    + ", pid " + Binder.getCallingPid());
        }
        if (account == null) throw new IllegalArgumentException("account is null");
        int userId = UserHandle.getCallingUserId();
        if (!isAccountManagedByCaller(account.type, callingUid, userId)) {
            String msg = String.format(
                    "uid %s cannot explicitly add accounts of type: %s",
                    callingUid,
                    account.type);
            throw new SecurityException(msg);
        }
        /*
         * Child users are not allowed to add accounts. Only the accounts that are
         * shared by the parent profile can be added to child profile.
         *
         * TODO: Only allow accounts that were shared to be added by
         *     a limited user.
         */

        // fails if the account already exists
        long identityToken = clearCallingIdentity();
        try {
            UserAccounts accounts = getUserAccounts(userId);
            return addAccountInternal(accounts, account, password, extras, callingUid);
        } finally {
            restoreCallingIdentity(identityToken);
        }
    }
```

> `Binder.getCallingUid();` 获取当前用户，binder 的一大优势，相对Linux内核提供的多进程访问方式，Binder可以严格区分用户
>
> `UserHandle.getCallingUserId()`获取当前需要新用户可以分配的id，在/data/system/sync 目录下可查看
>
> `clearCallingIdentity` 重置一个当前线程到来的IPC的标识
>
> `getUserAccounts`获取账户相关存储工具
>
> * 如果是不存在的用户会根据用户id创建两个数据库文件，并且创建UserAccounts对象
> * `getPreNDatabaseName` 在 （/data/system/accounts.db  旧的目录中的数据库 会自动 merge到新的目录----只针对新用户） /data/users/{userid}/accounts.db
> * `getDeDatabaseName` 在 /data/system/{userid}/accounts_de.db
> * 在创建UserAccounts的时候回自动将accounts.db(旧)合并数据到account_de.db中，此时内存SparseArray会自动同步一份
> * 如果当前创建的UserAccounts是新创建的，此时会删除（检索当前应用安装包不存在时）应用存储信息