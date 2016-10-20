---
title: 自定义可扩展式ExpandableRecyclerVIew
categories:
  - Android
id: 218
date: 2016-06-15 09:30:34
tags:
---

**RecyclerView**是个万花筒，比起ListView好用很多，项目需要用到可扩展的列表用来显示每一项的更多信息，Github上这样的类库有很多，无非有两种形式，第一种是在点击一个Item的时候在该Item下方添加一个子布局并且添加动画，RecyclerView的刷新函数`notifyItemInserted(position)`自带默认的插入动画。第二种就是在点击一个Item的时发返回不同的**ItemViewType**，根据不同**ItemViewType**加载不同的布局。本人是用的第二种方式实现的，有时间去亲自写一下第一种。

* * *

首先我们要设计好折叠和扩展下的不同布局,具体布局由于分开了设计有好几个文件所以只是给出布局名字，收缩时的布局为**R.layout.alarm_list_item_collapsed**，扩展时的布局为**R.layout.alar_list_item_expanded**。

接下来我们来到ViewHolder,由于折叠状态和展开状态有公共的业务逻辑，所以我写了一个ViewHolder抽象父类叫做AlarmViewHolder，下面是主要的核心代码，省略了部分业务逻辑

```java
public abstract class AlarmViewHolder extends RecyclerView.ViewHolder {

    public Alarm alarm;
    //该控件为折叠展开都具有的控件
    public ImageButton mArrow;

    public AlarmViewHolder(View itemView) {
        super(itemView);
        mArrow = (ImageButton)itemView.findViewById(R.id.arrpw);
        //折叠展开公共控件事件
        setListeners();
    }

    public void setData(Alarm alarm){
        this.alarm =alarm;
    }

    public void clearData(){
        alarm =null;
    }
    //绑定alarm的信息到UI界面
    public abstract void bindAlarm(Alarm alarm);
}
```

### 接下来继承上面的父类编写折叠状态时的ViewHolder

```java
/**
 * 折叠状态的ViewHolder
 */
public class AlarmCollapsedViewHolder extends AlarmViewHolder {

    private final AlarmAdapter mAlarmAdapter;
    private final Context mContext;

    public AlarmCollapsedViewHolder(View itemView, AlarmAdapter alarmAdapter) {
        super(itemView, alarmClickHandler);
        mContext = itemView.getContext();
        mAlarmAdapter = alarmAdapter;
        //扩展
        mArrow.setOnClickListener(v -&gt; {
            mAlarmAdapter.expand(getAdapterPosition());
        });
        setListeners();
    }
    @Override
    public void bindAlarm(Alarm alarm) {
        setData(alarm);
        //在此绑定响应数据
    } 
}
```

先提前说下后面会提到的**AlarmAdapter**，Adapter里包涵有扩展和折叠的方法，在这里用到了里面的`expand()`方法就是扩展的方法。**AlarmCollapsedViewHolder**继承AlarmViewHolder实现bindAlarm方法进行绑定alarm数据到UI界面，公共控件（应该说是同一个id）mArrow在这里是可以触发扩展事件。

### 下面该实现扩展的ViewHolder了，跟折叠的ViewHolder差不多

```java
public class AlarmExpandedViewHolder extends AlarmViewHolder {

    private final AlarmAdapter mAlarmAdapter;
    private final Context mContext;

    public AlarmExpandedViewHolder(View itemView, AlarmAdapter alarmAdapter) {
        super(itemView, alarmClickHandler);
        mContext = itemView.getContext();
        mAlarmAdapter = alarmAdapter;
         //折叠Edit
        mArrow.setOnClickListener(v -&gt; mAlarmAdapter.collapse(getAdapterPosition()));
    }

    @Override
    public void bindAlarm(Alarm alarm) {
        setData(alarm);

    }
}
```
与折叠ViewHolder的唯一区别就在于**mArrow**的点击事件触发的是收缩事件。</span></span>

### 为了展开之后能够使item定位到合适的位置定义一个接口

```java
public interface IRecyclerViewScroll {

    /**
     *设置滚动到相应item的Id
     */
    void setSmoothScrollStableId(String stableId);

    /**
     * 滚动到对应的位置
     */
    void smoothScrollTo(int position);
}
```

### 到这里就该写上面提到的AlarmAdaper了

```java
public class AlarmAdapter extends RecyclerView.Adapter<AlarmViewHolder> {

    private static final int VIEW_TYPE_ALARM_COLLAPSED = R.layout.alarm_list_item_collapsed;
    private static final int VIEW_TYPE_ALARM_EXPANDED = R.layout.alarm_list_item_expanded;

    private int mExpandedPosition = -1;
    private String mExpandedId = Alarm.INVALID_ID;
    private IRecyclerViewScroll mIRecyclerViewScroll;
    private Context mContext;
    private List<Alarm> mList = new ArrayList<>();

    public AlarmAdapter(Context context, IRecyclerViewScroll iRecyclerViewScroll) {
        mIRecyclerViewScroll = iRecyclerViewScroll;
        mContext = context;
        setHasStableIds(true);
    }

    @Override
    public AlarmViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(mContext).inflate(viewType, parent, false);
        if (viewType == VIEW_TYPE_ALARM_COLLAPSED) {
            return new AlarmCollapsedViewHolder(view, this, mAlarmClickHandler);
        } else {
            return new AlarmExpandedViewHolder(view, this, mAlarmClickHandler, mAlarmUpdateHandler);
        }
    }

    @Override
    public void onBindViewHolder(AlarmViewHolder holder, int position) {
        Alarm alarm = mList.get(position);
        holder.bindAlarm(alarm);
    }

    @Override
    public void onViewRecycled(AlarmViewHolder holder) {
        super.onViewRecycled(holder);
        holder.clearData();
    }

    /**
     * 避免notifyDataSetChanged()之后相同位置返回不同的对象，导致notifyDataSetChanged()
     * 之后视图刷新不稳定，出现闪烁。
     * 由于主键为UUID，所以取整型的hashCode作为UUID的代替唯一标识
     *
     * @param position
     * @return
     */
    @Override
    public long getItemId(int position) {
        //此处返回对应的AlarmId,主要用于对比滑动Id
        Alarm alarm = mList.get(position);
        if (alarm == null) {
            return RecyclerView.NO_ID;
        }
        UUID uuid = UUID.fromString(alarm.getId());
        return uuid.hashCode();
    }

    /**
     * 返回Item的Id也即UUID
     *
     * @param position
     * @return
     */
    public String getItemUUID(int position) {
        //此处返回对应的AlarmId,主要用于对比滑动Id
        Alarm alarm = mList.get(position);
        if (alarm == null) {
            return "-1";
        }
        return alarm.getId();
    }

    @Override
    public int getItemCount() {
        return mList.size();
    }

    @Override
    public int getItemViewType(int position) {
        //将布局当作viewtype
        final String stableId = getItemUUID(position);
        return stableId != String.valueOf(RecyclerView.NO_ID) && stableId.equals(mExpandedId)
                ? VIEW_TYPE_ALARM_EXPANDED : VIEW_TYPE_ALARM_COLLAPSED;

    }

    /**
     * 扩展并且滑动到这个item
     */
    public void expand(int position) {
        final String stableId = getItemUUID(position);
        //当前Id和扩展Id一样无须再次扩展
        if (mExpandedId.equals(stableId)) {
            return;
        }
        mExpandedId = stableId;
        mIRecyclerViewScroll.smoothScrollTo(position);
        if (mExpandedPosition != 0) {
            notifyItemChanged(mExpandedPosition);
        }
        mExpandedPosition = position;
        notifyItemChanged(position);
    }

    //收缩
    public void collapse(int position) {
        mExpandedId = Alarm.INVALID_ID;
        mExpandedPosition = -1;
        notifyItemChanged(position);
    }

    //更新数据
    public void setData(List<Alarm> list) {
        if (list != null) {
            mList.clear();
            mList.addAll(list);
        }
        notifyDataSetChanged();
    }

    public String getmExpandedId() {
        return mExpandedId;
    }

    public void setmExpandedId(String mExpandedId) {
        this.mExpandedId = mExpandedId;
    }
}
```
这个类的内容比较多，只是贴上了核心代码，整体逻辑还是比较清晰的。先声明该项目的数据库主键使用的字符串类型的UUID，所以重写**getItemId**的时候返回的是UUID的Hash值（数据库为整型的话直接返回数据库Id就行了），此方法必须重写而且返回唯一值，并且需要声明`setHasStableId(true)`，主要是为了给每一个Item一个唯一Id,这样在扩展的时候不容易乱掉。

接下来我们在`getViewType()`里根据不同条件返回不同的布局，返回扩展布局的条件是当前id不等于RecyclerView的默认空Id并且当前Id要等于**mExpandId**，我们可以看到在`expand()`方法里我们对**mExpandId**进行的对比和赋值然后刷新布局，在`collapse()`方法里讲**mExpandId**设置为初始值（-1）,这样就可以达到折叠效果。

应该可以注意到在`expand()`方法里调用了**IRecyclerViewScroll**接口的`smoothScroll()`方法，为什么要在这里调用它，看下这个函数的意思就知道，我们会在他的回调方法里去定位到扩展Item。

### 在用到RecyclerView的Activity或者Fragment里使用

```java
public class AlarmListFragment extends Fragment implements IRecyclerViewScroll,
        LoaderManager.LoaderCallbacks<List<Alarm>>{

    public static final String EXTRA_SCROLL_TO_ALARM = "extrascroll_to_alarm";

    private RecyclerView mRecyclerView;
    private AlarmAdapter mAlarmAdapter;
    private String mScrollId;
    private String mExpandId = Alarm.INVALID_ID;
    private Loader mLoader;
    //与Activity交互接口
    private IAlarmListFragmentListener mListener;

    public AlarmListFragment() {
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setHasOptionsMenu(true);
        //初始化loader
        mLoader = getLoaderManager().initLoader(0, null, this);

    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_alarm_list, container, false);
    }

    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        Toolbar toolbar = (Toolbar) view.findViewById(R.id.toolbar);
        ((AppCompatActivity) getActivity()).setSupportActionBar(toolbar);
        mLinearLayoutManager = new LinearLayoutManager(getActivity());
        initViewValue();
    }

    @Override
    public void onResume() {
        super.onResume();
        final Intent intent = getActivity().getIntent();
        if (intent.hasExtra(EXTRA_SCROLL_TO_ALARM)) {
            String alarmId = intent.getStringExtra(Alarm.EXTRA_ALARM_ID);
            if (!TextUtils.isEmpty(alarmId) && !alarmId.equals(Alarm.INVALID_ID)) {
                setSmoothScrollStableId(alarmId);
                mLoader.forceLoad();
            }
        }
    }

    private void initViewValue() {
        mScrollId = Alarm.INVALID_ID;
        mRecyclerView.setLayoutManager(mLinearLayoutManager);
        mAlarmAdapter.setmExpandedId(mExpandId);
        mRecyclerView.setAdapter(mAlarmAdapter);
    }

    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        if (context instanceof IAlarmListFragmentListener) {
            mListener = (IAlarmListFragmentListener) context;
        }
    }

    @Override
    public Loader<List<Alarm>> onCreateLoader(int id, Bundle args) {
        Log.d(TAG, "onCreateLoader: ");
        return new AlarmAsyncTaskLoader(getActivity());
    }

    @Override
    public void onLoadFinished(Loader<List<Alarm>> loader, List<Alarm> data) {
        Log.d(TAG, "onLoadFinished: ");
        mAlarmAdapter.setData(data);
        if (!mScrollId.equals(Alarm.INVALID_ID)) {
            scrollToPosition();
            setSmoothScrollStableId(Alarm.INVALID_ID);
        }

    }

    /**
     * 更新数据之后滑动
     */
    private void scrollToPosition() {
        int alarmCount = mAlarmAdapter.getItemCount();
        for (int position = 0; position < alarmCount; position++) {
            if (mScrollId.equals(mAlarmAdapter.getItemUUID(position))) {
                mAlarmAdapter.expand(position);
                break;
            }
        }
    }
    @Override
    public void onLoaderReset(Loader<List<Alarm>> loader) {
        mAlarmAdapter.setData(null);
    }

    @Override
    public void setSmoothScrollStableId(String stableId) {
        mScrollId = stableId;
    }
    @Override
    public void smoothScrollTo(int position) {
        mLinearLayoutManager.scrollToPositionWithOffset(position, 20);
    }
}
```

在这个Fragment里 我使用的**AsyncTaskLoader**加载数据，`onLoadFinish()`里调用了`setSmoothScrollStableId(Alarm.INVALID_ID)`方法进行初始化设置，加载完成是不会扩展的，注意下`onResume()`里我们也调用`setSmoothScrollStableId(alarmId)`，其实在第一次打开APP的时候**alarmId**是空的所以不会扩展，但是在这里调用的目的可以根据**Intent**看到是别的页面跳转过来，指定闹钟扩展。

接下来看前面提到的`smoothScrollTo()`，我们在这里执行了`mLinearLayoutManager.scrollToPositionWithOffest(position,20)`,这里我们用这个方法跳到指定的Item。
这样RecyclerView就实现了完美的扩展。

