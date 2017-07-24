---
layout: post
title: Android ContentProvider
categories: Android
description: Android ContentProvider
keywords: Andorid, Android ContentProvider
---

## ContentProvider概述
ContentProvider是Android四大组件之一，为存储和获取数据提供统一的接口，可以在不同的应用之间共享数据。数据的存储有很多中方式，比如SQLite和XML文件方式，但在不同的应用程序中，数据是不能直接被相互访问和操作的。
ContentProvider分为系统和自定义的，系统的如联系人、图片、音频、视频等
ContentProvider使用表来组织数据，提供方法如下：
- public boolean onCreate() ：在创建ContentProvider时调用，只有在ContentProvider被第一次使用时才会被调用创建
- public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) ：用于查询指定Uri的ContentProvider，返回一个Cursor。其中Projection是需要从ContentProvider中选择的字段（列映射别名，可以一致），如果为空，则返回所有的字段，sortOrder为默认的排序规则，其余参数同delete方法
- public Uri insert(Uri uri, ContentValues values) ：用于添加数据到指定Uri的ContentProvider中，values为需要添加数据的键值对
- public int delete（Uri uri, String selection, String[] selectionArgs)：用于删除指定URI的ContentProvider中的数据，其中selection为构成筛选添加的语句，如“id=1”或者“id=？”，selectionArgs为对应selection的两种情况传入null，或者new String[]{"1"},当然这两个参数都可以为null，可以由前面的URI指定对应id的数据。
- public int update（Uri uri， ContentValues values, String selection, String[] selectionArgs)：对 ContentProvider中的数据进行更新。values为需要更新的内容键值对（键为对应的字段，值为修改值）。
- public String getType（Uri uri）：返回当前Uri所代表数据的MIME类型，如果要数据的类型属于集合类型，则MIME类型字符串应该以vnd.android.cursor.dir/开头，否则应该以vnd.android.cursor.item/，可具体看后面的代码实例。

## Uri
可以看到在ContentProvider中，对于数据的操作都是通过一个叫Uri的参数，所以先来看Uri是什么。
uri为通用资源标识符，代表要操作的数据，Android上可用的每种资源（图像、视频等）都可以用Uri来表示。它主要由三个部分组成:"content://"、数据的路径、标识ID（可选）(content://com.asus.MyContentProvider/user/1)。这个Uri可以和Http结合起来理解：http://www.xxx.com/xxx/12
- content://：android定义的标准，类似于http://
- com.asus.MyContentProvider：该contentProvider的唯一标识，外部调用者通过这个authority找到它
- user/1：路径，代表某个具体需要操作的数据位置

既然说到这个Uri，就需要介绍另外两个和Uri相关的操作类：**UriMatcher**和**ContentUris**
- UriMatcher
在一个数据库中，可以包含多个表，使用UriMatcher可以帮助我们方便的过滤到某一张表。
具体用法如下：
1、初始化，常量UriMatcher.NO_MATCH表示不匹配任何路径的返回码
```
UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH);
```
2、注册需要的Uri
```
matcher.addURI(MyUser.AUTHORITY,"/user",MY_USERS);
matcher.addURI(MyUser.AUTHORITY,"/user/#",MY_USER_SINGLE);
```
3、与注册的Uri匹配过滤
```
        int match = matcher.match(uri);
        switch (match){
            case MY_USERS:
                return MyUser.User.CONTENT_TYPE;
            case MY_USER_SINGLE:
                return MyUser.User.CONTENT_TYPE_ITEM;
            default:
                throw  new UnsupportedOperationException("Not support");
        }
```

- ContentUris
该类用于在Uri后加上ID和获取Uri路径后面的ID，用法如下：
在Uri后增加id
```
Uri uri = ContentUris.withAppendedId(MyUser.User.CONTENT_URI,uid);
```
获取Uri后增加的id
```
long id = ContentUris.parseId(uri);
```
>和这个类似的还有一个函数：getPathSegments，该函数是将Uri的path部分（第三部分）拆分，去掉"/",如Uri为content://com.asus.MyContentProvider/user/1，则通过uri.getPathSegments().get(1)获取的则是"1"

## 自定义contentProvider
下面就具体来看怎么自定义一个contentProvider
1、首先定义一个自己的数据结构，用于方便的记录更改根数据库相关的信息，公开的数据中，一定要有个_ID字段，因为在Android中，数据的存储方式是以一个表格的形式存储的，_ID字段唯一标示了每项数据，所以常规做法是将我们的类继承自BaseColumns类：
```
package com.asus.mycontentprovider;

import android.net.Uri;
import android.provider.BaseColumns;

/**
 * Created by root on 17-7-20.
 */

public class MyUser {
    public static final String AUTHORITY = "com.asus.MyContentProvider";

    //表名
    public static final String USERS_TABLE_NAME = "MyUser";

    //数据库的版本
    public static final int DATABASE_VERSION = 1;

    //实现BaseColumns接口以后，内部类就可以继承一个主键字段_ID,这也是很多Android里的类所需要的.
    //BaseColumns接口有两个常量：1.总行数_count   2.每行的独特的_id
    public static final class User implements BaseColumns {

        //访问该ContentProvider的URI，唯一的字符串值，最好的方案是以类的全名称
        public static final Uri CONTENT_URI = Uri.parse("content://"+AUTHORITY+"/user");

        //该ContentProvider所返回的数据类型的定义
        public static final String CONTENT_TYPE = "vnd.android.cursor.dir/vnd.myprovider.user";
        public static final String CONTENT_TYPE_ITEM = "vnd.android.cursor.item/vnd.myprovider.user";

        //表数据列，必须为其定义一个叫_id的列，它用来表示每条记录的唯一性
        public static final String _ID = "_id";
        public static final String NAME = "name";
        public static final String AGE = "age";
    }
}
```
2、创建一个数据源，即数据存储系统，我们一般是是使用SQLite数据库，为了方便操作数据库，我们继承SQLiteOpenHelper辅助类：
```
private static class DatabaseHelper extends SQLiteOpenHelper {
        DatabaseHelper(Context context) {
            super(context, MyUser.USERS_TABLE_NAME, null, MyUser.DATABASE_VERSION);
        }

        @Override
        public void onCreate(SQLiteDatabase sqLiteDatabase) {
            String sql = "CREATE TABLE IF NOT EXISTS MyUser"+
                    "(_id INTEGER PRIMARY KEY AUTOINCREMENT, name VARCHAR, age INTEGER)";
            sqLiteDatabase.execSQL(sql);
        }

        @Override
        public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {
            String sql = "DROP TABLE IF EXISTS MyUser";
            sqLiteDatabase.execSQL(sql);
            onCreate(sqLiteDatabase);
        }
    }
```

3、创建一个继承自ContentProvider的类，并重写insert、delete、query、update、getType、onCreate方法，在这些方法中实现对数据源的操作：
```
public class MyContentProvider extends ContentProvider{
    //访问表的所有列
    public static final int MY_USERS = 1;
    //访问单独的列
    public static final int MY_USER_SINGLE = 2;

    private static UriMatcher matcher;

    private static HashMap<String,String> userMap;

    static{
        matcher = new UriMatcher(UriMatcher.NO_MATCH);
        matcher.addURI(MyUser.AUTHORITY,"/user",MY_USERS);
        matcher.addURI(MyUser.AUTHORITY,"/user/#",MY_USER_SINGLE);

        //setProjectionMap使用 这个函数的作用是在qb里设置数据库字段的别名，即用户定义列名->数据库列名
        //在执行qb.query时可以使用用户定义列名设置projection、selection等
        //如果不需要对列名进行映射，可以不调用这个函数
        //但是如果调用，必须对所有列都进行映射，映射中key和value可以完全相同
        userMap = new HashMap<String,String>();
        userMap.put(MyUser.User._ID,MyUser.User._ID);
        userMap.put(MyUser.User.NAME,MyUser.User.NAME);
        userMap.put(MyUser.User.AGE,MyUser.User.AGE);
    }

    private static class DatabaseHelper extends SQLiteOpenHelper {
        DatabaseHelper(Context context) {
            super(context, MyUser.USERS_TABLE_NAME, null, MyUser.DATABASE_VERSION);
        }

        @Override
        public void onCreate(SQLiteDatabase sqLiteDatabase) {
            String sql = "CREATE TABLE IF NOT EXISTS MyUser"+
                    "(_id INTEGER PRIMARY KEY AUTOINCREMENT, name VARCHAR, age INTEGER)";
            sqLiteDatabase.execSQL(sql);
        }

        @Override
        public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {
            String sql = "DROP TABLE IF EXISTS MyUser";
            sqLiteDatabase.execSQL(sql);
            onCreate(sqLiteDatabase);
        }
    }
    private DatabaseHelper dh;
    private SQLiteDatabase mUserDB;
    @Override
    public boolean onCreate() {
        dh = new DatabaseHelper(getContext());
        mUserDB = dh.getWritableDatabase();
        return (mUserDB == null) ? false : true;
    }

    @Nullable
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        int match = matcher.match(uri);
        SQLiteQueryBuilder sqLiteQueryBuilder = new SQLiteQueryBuilder();
        sqLiteQueryBuilder.setTables(MyUser.USERS_TABLE_NAME);
        sqLiteQueryBuilder.setProjectionMap(userMap);

        if(match == MY_USER_SINGLE) {
            long id = ContentUris.parseId(uri);
            sqLiteQueryBuilder.appendWhere("_id = "+ id);
        }

        Cursor c = sqLiteQueryBuilder.query(mUserDB,projection,selection,selectionArgs,null,null,sortOrder);
        c.setNotificationUri(getContext().getContentResolver(), uri);
        return c;
    }

    @Nullable
    @Override
    public String getType(Uri uri) {
        int match = matcher.match(uri);
        switch (match){
            case MY_USERS:
                return MyUser.User.CONTENT_TYPE;
            case MY_USER_SINGLE:
                return MyUser.User.CONTENT_TYPE_ITEM;
            default:
                throw  new UnsupportedOperationException("Not support");
        }
    }

    @Nullable
    @Override
    public Uri insert(Uri uri, ContentValues contentValues) {
        Log.d("chris","insert");
        long rowId = mUserDB.insert(MyUser.USERS_TABLE_NAME,null,contentValues);
        if(rowId > 0) {
            Log.d("chris","insert success rowId="+rowId);
            Uri insertUri = ContentUris.withAppendedId(MyUser.User.CONTENT_URI,rowId);
            notifyDataChanged(insertUri);
            return insertUri;
        }
        return null;
    }

    //删除数据后，数据表ID并不会自动更新一次，仍是保持原来的id
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        int match = matcher.match(uri);
        int count = 0;
        switch (match) {
            case MY_USERS:
                Log.d("chris","delete all");
                count = mUserDB.delete(MyUser.USERS_TABLE_NAME,selection,selectionArgs);
                break;
            case MY_USER_SINGLE:
                //String id = uri.getPathSegments().get(1);
                long id = ContentUris.parseId(uri);
                Log.d("chris","delete id = "+id);
                count = mUserDB.delete(
                        MyUser.USERS_TABLE_NAME,
                        "_id = " + id + (!TextUtils.isEmpty(selection) ? " AND (" +
                        selection + ")" : ""), selectionArgs);
                break;
            default:
                throw new IllegalArgumentException("chris Unknown uri");

        }

        Log.d("chris","delete count="+count);
        if(count > 0) {
            notifyDataChanged(uri);
        }
        return count;
    }

    @Override
    public int update(Uri uri, ContentValues contentValues, String selection, String[] selectionArgs) {
        int match = matcher.match(uri);
        int count = 0;
        switch (match) {
            case MY_USERS:
                count = mUserDB.update(MyUser.USERS_TABLE_NAME,contentValues,selection,selectionArgs);
                break;
            case MY_USER_SINGLE:
                long id = ContentUris.parseId(uri);
                count = mUserDB.update(
                        MyUser.USERS_TABLE_NAME,
                        contentValues,"_id = " + id + (!TextUtils.isEmpty(selection) ? " AND (" +
                                selection + ")" : ""), selectionArgs);
                break;
            default:
                throw new IllegalArgumentException("Unknow URI");
        }
        if(count > 0) {
            notifyDataChanged(uri);
        }
        return count;
    }

    //通知指定URI数据已改变
    private void notifyDataChanged(Uri uri) {
        getContext().getContentResolver().notifyChange(uri,null);
    }
}
```

4、在AndroiManifest声明该contentProvider，需要注意的是authorities需要和MyUser中定义的AUTHORITY需要保持一致，不然会报找不到该ContentProvider的错误
```
        <provider
            android:authorities="com.asus.MyContentProvider"
            android:name=".MyContentProvider"
            />
```

5、使用自定义的ContentProvider，为了方便操作contentProvider，Android系统为我们提供了ContentResolver类，可以使用getContentProvider（）方法返回一个ContentResolver对象，并使用它与ContentProvider对应的方法来操作。
```
public class MainActivity extends AppCompatActivity implements View.OnClickListener{
    private ContentResolver mContentResolver;

    private Button insert;
    private Button delete;
    private Button query;
    private Button update;
    private TextView textView;
    private EditText name;
    private EditText id;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mContentResolver = getContentResolver();
        initView();
    }

    public void initView() {
        insert = (Button)findViewById(R.id.insert);
        insert.setOnClickListener(this);
        delete = (Button)findViewById(R.id.delete);
        delete.setOnClickListener(this);
        query = (Button)findViewById(R.id.query);
        query.setOnClickListener(this);
        update = (Button)findViewById(R.id.update);
        update.setOnClickListener(this);
        textView = (TextView) findViewById(R.id.text);
        name = (EditText) findViewById(R.id.name);
        id = (EditText) findViewById(R.id.id);
    }

    @Override
    public void onClick(View view) {
        switch (view.getId()){
            case R.id.insert:
                insert();
                break;
            case R.id.delete:
                delete();
                break;
            case R.id.query:
                query();
                break;
            case R.id.update:
                update();
                break;
            default:
                //do noting
        }
    }

    private void insert() {
        ContentValues values = new ContentValues();
        String mName = name.getText().toString();
        values.put(MyUser.User.NAME,mName);
        values.put(MyUser.User.AGE,25);
        mContentResolver.insert(MyUser.User.CONTENT_URI,values);
    }

    private void delete() {
        String mId = id.getText().toString();
        int uid = Integer.parseInt(mId);
        Uri uri = ContentUris.withAppendedId(MyUser.User.CONTENT_URI,uid);
        mContentResolver.delete(uri,null,null);
        //如前面所述，也可以调用条件来删除指定条件的数据
        //mContentResolver.delete(uri,"name="+"chris",null)
    }

    private void query() {
        String[] PROJECTION = new String[] {
                MyUser.User._ID,
                MyUser.User.NAME,
                MyUser.User.AGE
        };
        Cursor cursor = mContentResolver.query(MyUser.User.CONTENT_URI,PROJECTION,null,null,null);
        if(cursor != null) {
            Log.d("chris","cursor != null "+cursor.getCount());
            if (cursor.moveToFirst()) {
                for (int i = 0; i < cursor.getCount(); i++) {
                    cursor.moveToPosition(i);
                    String name = cursor.getString(1);
                    int age = cursor.getInt(2);
                    Log.d("chris", "query name:" + name + " age:" + age);
                }
            }
        } else {
            Log.d("chris","cursor == null");
        }
    }

    private void update() {
        String mId = id.getText().toString();
        int uid = Integer.parseInt(mId);
        String mName = name.getText().toString();
        Uri uri = ContentUris.withAppendedId(MyUser.User.CONTENT_URI,uid);
        ContentValues values = new ContentValues();
        values.put(MyUser.User.NAME,mName);
        values.put(MyUser.User.AGE,25);
        mContentResolver.update(uri,values,null,null);
    }
}

```
