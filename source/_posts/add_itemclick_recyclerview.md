---
title: 为RecyclerView添加ItemClick点击事件
categories:
  - Android
id: 53
date: 2015-12-01 23:52:14
tags:
---

RecyclerView是V7包下的一个新控件，功能强大，自定义度高，可以说可以代替listview，gridview!由于高度的可自定义，RecyclerView并没有提供像ListView那样的Item点击事件，这样只能自己想办法。

### 定义一个接口

```java
public interface ItemClickListener {

     void onItemClick(Item item);
}
```
### 继承RecyclerView.Adapter.ViewHolder重写Adapter

```java
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.myViewHolder> {

    private Context mContext;
    private ItemClickListener mListener;

    public MainGoodsAdapter(Context context,ItemClickListener listener) {
        this.mContext = context;
        this.mListener=listener;
    } @Override
    public myViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(mContext).inflate(R.layout.xx, parent, false);
        return new myViewHolder(view);
    }

    @Override
    public void onBindViewHolder(myViewHolder holder, int position) {

        holder.mIem = mList.get(position);
        holder.mView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mListener.onItemClick(holder.mIem );
            }
        });
        }
    }
    class myViewHolder extends RecyclerView.ViewHolder {

        public View mView;
        public Item mIem;

        public myViewHolder(final View itemView) {
            super(itemView);
            mView=itemView;
        }
    }
}
```
### 在对应的显示页面继承这个接口，然后实现这个接口，重写回调函数

```java
@Override
public void onItemClick(View itemView, int position) {

}
```
对于第二部中把传入接口的参数放到Adapter的构造方法中只是一种方法，也可以自定义一个函数，来传参，并传递给Adapter的全局变量即可。


