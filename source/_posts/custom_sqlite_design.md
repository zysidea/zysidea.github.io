---
title: Android数据库方法封装
tags:
  - Android
  - SQLite
categories:
  - Android
id: 184
date: 2016-05-09 13:41:19
---

对于复杂的数据，一般来说ContentProvider是个不错的选择，但是有时候我们并没有必要去共享数据，在比较简单的数据</span></span>情况下，下面这种封装方法非常好用
### 先定义一个bean，方便起见，只有id，hour，minutes三个字段

```java
public class Alarm implements Parcelable {

    private String id;
    private int hour;
    private int minutes;

    public Alarm() {
    }

    public Alarm(int hour, int minute) {
        this.id= UUID.randomUUID().toString().toUpperCase();
        this.hour =hour;
        this.minutes=minute;
    }

    public Alarm(Cursor cursor) {
        this.id = cursor.getString(AlarmContract.ID_INDEX);
        this.hour = cursor.getInt(AlarmContract.HOUR_INDEX);
        this.minutes = cursor.getInt(AlarmContract.MINUTES_INDEX);
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public int getHour() {
        return hour;
    }

    public void setHour(int hour) {
        this.hour = hour;
    }

    public int getMinutes() {
        return minutes;
    }

    public void setMinutes(int minutes) {
        this.minutes = minutes;
    }
    @Override
    public String toString() {
        return "Alarm{" +
                "id=" + id +
                ", hour=" + hour +
                ", minutes=" + minutes +
                '}';
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.id);
        dest.writeInt(this.hour);
        dest.writeInt(this.minutes);
    }

    protected Alarm(Parcel in) {
        this.id = in.readString();
        this.hour = in.readInt();
        this.minutes = in.readInt();
    }

    public static final Creator<Alarm> CREATOR = new Creator<Alarm>() {
        @Override
        public Alarm createFromParcel(Parcel source) {
            return new Alarm(source);
        }

        @Override
        public Alarm[] newArray(int size) {
            return new Alarm[size];
        }
    };
    
```
### 定义数据库字段支持类，为数据库提供常量

```java
public class AlarmContract {

    public final static String DB_NAME = "alarms.db";
    public final static String TABLE_ALARMS_NAME = "alarms";
    public final static int DB_VERSION = 1;

    //数据库中字段个数
    public static final int VALUSE_SIZE = 12;
    //cursor中index
    public static final int ID_INDEX = 0;
    public static final int HOUR_INDEX = 1;
    public static final int MINUTES_INDEX = 2;

    public interface AlarmsColumns extends BaseColumns {

        //小时
        String HOUR = "hour";
        //分钟
        String MINUTE = "minute"
    }
}
```
### 创建DataBaseOpenHelper创建本地数据库
```java
public class AlarmDataBaseOpenHelper extends SQLiteOpenHelper {

    private static AlarmDataBaseOpenHelper mDataBaseOpenHelper;

    // 创建alarms表
    private static void createAlarmTable(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE " + AlarmContract.TABLE_ALARMS_NAME + "(" +
                AlarmContract.AlarmsColumns._ID + " TEXT PRIMARY KEY NOT NULL , " +
                AlarmContract.AlarmsColumns.HOUR + " INTEGER NOT NULL, " +
                AlarmContract.AlarmsColumns.MINUTES + " INTEGER NOT NULL);");

    }

    public AlarmDataBaseOpenHelper(Context context) {
        super(context, AlarmContract.DB_NAME, null, AlarmContract.DB_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        try {
            createAlarmTable(db);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

    }

    public static synchronized AlarmDataBaseOpenHelper getInstance(Context context) {
        if (mDataBaseOpenHelper == null) {
            mDataBaseOpenHelper = new AlarmDataBaseOpenHelper(context);
        }
        return mDataBaseOpenHelper;
    }
  ```
### 创建数据库工具类，用于打开数据库
```java
public class AlarmDataBase {

    private AlarmDataBaseOpenHelper mDbHelper;
    private final Context mContext;
    public static AlarmDAO mAlarmDAO;

    public AlarmDataBase(Context context) {
        this.mContext = context;
    }

    /**
     * 打开数据库连接并且获得DAO
     */
    public void open() {
        try {
            mDbHelper = AlarmDataBaseOpenHelper.getInstance(mContext);
            if (mAlarmDAO == null) {
                SQLiteDatabase db = mDbHelper.getWritableDatabase();
                mAlarmDAO = new AlarmDAO(db);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }

    }
}
```
### 创建DAO层抽象父类，提供共有的基本CRUD
```java
public abstract class DataBaseProvider {

    private SQLiteDatabase mDb;

    public DataBaseProvider(SQLiteDatabase db) {
        mDb = db;
    }
    public Cursor query(String tableName, String[] columns,
                        String selection, String[] selectionArgs, String sortOrder) {

        final Cursor cursor = mDb.query(tableName, columns,
                selection, selectionArgs, null, null, sortOrder);

        return cursor;
    }

    public Cursor query(String tableName, String[] columns,
                        String selection, String[] selectionArgs, String sortOrder, String limit) {
        return mDb.query(tableName, columns, selection,
                selectionArgs, null, null, sortOrder, limit);
    }

    public Cursor query(String tableName, String[] columns,
                        String selection, String[] selectionArgs, String groupBy,
                        String having, String orderBy, String limit) {

        return mDb.query(tableName, columns, selection,
                selectionArgs, groupBy, having, orderBy, limit);
    }

    public Cursor rawQuery(String sql, String[] selectionArgs) {
        return mDb.rawQuery(sql, selectionArgs);
    }

    public long insert(String tableName, ContentValues values) {
        return mDb.insert(tableName, null, values);
    }

    public int delete(String tableName, String selection, String[] selectionArgs) {
        return mDb.delete(tableName, selection, selectionArgs);
    }

    public int update(String tableName, ContentValues values, String whereClause, String[] whereArgs) {
        return mDb.update(tableName, values, whereClause, whereArgs);
    }
}
```
### 定义对应数据库表Alarm的操作接口，添加数据库操作均在此定义
```java
public interface IAlarmDAO {

    //添加闹钟
    long addAlarm(Alarm alarm);

    //删除闹钟
    int deleteAlarmById(String alarmId);

    //修改闹钟
    int updateAlarmById(Alarm alarm);

    //获取所有的闹钟
    List<Alarm> fetchAllAlarm();

    //根据Id获取闹钟
    Alarm fetchAlarmById(String alarmId);
}
```
### 定义AlarmDAO类，具体化数据库操作
```java
public class AlarmDAO extends DataBaseProvider implements IAlarmDAO {

    public AlarmDAO(SQLiteDatabase db) {
        super(db);
    }

    /**
     * 添加一个Alarm
     *
     * @param alarm
     * @return
     */
    @Override
    public long addAlarm(Alarm alarm) {
        try {
            return super.insert(AlarmContract.TABLE_ALARMS_NAME, createContentValues(alarm));
        } catch (Exception e) {
            return -1;
        }

    }

    /**
     * 删除相应Id的Alarm
     *
     * @param alarmId
     * @return
     */
    @Override
    public int deleteAlarmById(String alarmId) {
        try {
            String selection = AlarmContract.AlarmsColumns._ID + " = ? ";
            String selectionArgs[] = new String[]{alarmId};
            return super.delete(AlarmContract.TABLE_ALARMS_NAME, selection, selectionArgs);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
    /**
     * 更新对应Id的Alarm
     *
     * @param alarm
     * @return
     */
    @Override
    public int updateAlarmById(Alarm alarm) {
        try {
            String selection = AlarmContract.AlarmsColumns._ID + " = ? ";
            String selectionArgs[] = new String[]{String.valueOf(alarm.getId())};
            return super.update(AlarmContract.TABLE_ALARMS_NAME, createContentValues(alarm), selection, selectionArgs);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    @Override
    public Alarm fetchAlarmById(String alarmId) {
        Alarm alarm = new Alarm();
        String selection = AlarmContract.AlarmsColumns._ID + " = ? ";
        String selectionArgs[] = new String[]{alarmId};
        Cursor cursor = super.query(AlarmContract.TABLE_ALARMS_NAME, null, selection, selectionArgs, null);
        if (cursor != null) {
            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                alarm = new Alarm(cursor);
                cursor.moveToNext();
            }
            cursor.close();
        }
        return alarm;
    }

    @Override
    public List<Alarm> fetchAllAlarm() {
        List<Alarm> list = new ArrayList<>();
        Cursor cursor = super.query(AlarmContract.TABLE_ALARMS_NAME, null, null, null, null);
        Log.d("TAG",cursor==null?0+"":cursor.getCount()+"");
        if (cursor != null) {
            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                list.add(new Alarm(cursor));
                cursor.moveToNext();
            }
            cursor.close();
        }
        return list;
    }

    /**
     * 根据Alarm创建contentValues
     * @param alarm
     * @return
     */
    public static ContentValues createContentValues(Alarm alarm) {
        ContentValues values = new ContentValues(AlarmContract.VALUSE_SIZE);
        values.put(AlarmContract.AlarmsColumns.HOUR, alarm.getHour());
        values.put(AlarmContract.AlarmsColumns.MINUTES, alarm.getMinutes());
        return values;
    }
    ```
### 在Application中打开数据库

```java
public class MyApplication extends Application {

    public static String TAG =LucidDreamAlarm.class.getSimpleName();
    public static  AlarmDataBase mAlarmDataBase;

    @Override
    public void onCreate() {
        super.onCreate();
        mAlarmDataBase = new AlarmDataBase(this);
        mAlarmDataBase.open();
    }
}
```
### 使用例子

```java
long alarmId = AlarmDataBase.mAlarmDAO.addAlarm(alarm);
if (alarmId != 0) {
    // TODO: 2016/5/8 添加成功之后开启闹钟
}
```


