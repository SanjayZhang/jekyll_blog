---
layout: post
tags:
 - android
---

[TOC]

ViewHoldr优化
======
listview的优化,相信是一个常用的组件。不过用的时候，有个小问题，就是数据多的时候，滑动起来会一卡一卡，不是十分流畅。相信大家都做过了优化，由于findviewByid效率比
较低,这边建立ViewHolder进行缓存，这是一个典型的空间换时间。但是这边写的时候比较繁琐，这边介绍一下优雅的写法。
    
ViewHolder一般写法
--- 
com.example.android.apis.view中的List14：
```
	/**
	 * Make a view to hold each row.
	 *
	 * @see android.widget.ListAdapter#getView(int, android.view.View,
	 *      android.view.ViewGroup)
	 */
	public View getView(int position, View convertView, ViewGroup parent) {
	    // A ViewHolder keeps references to children views to avoid unneccessary calls
	    // to findViewById() on each row.
	    ViewHolder holder;
	
	    // When convertView is not null, we can reuse it directly, there is no need
	    // to reinflate it. We only inflate a new View when the convertView supplied
	    // by ListView is null.
	    if (convertView == null) {
	        convertView = mInflater.inflate(R.layout.list_item_icon_text, null);
	
	        // Creates a ViewHolder and store references to the two children views
	        // we want to bind data to.
	        holder = new ViewHolder();
	        holder.text = (TextView) convertView.findViewById(R.id.text);
	        holder.icon = (ImageView) convertView.findViewById(R.id.icon);
	
	        convertView.setTag(holder);
	    } else {
	        // Get the ViewHolder back to get fast access to the TextView
	        // and the ImageView.
	        holder = (ViewHolder) convertView.getTag();
	    }
	
	    // Bind the data efficiently with the holder.
	    holder.text.setText(DATA[position]);
	    holder.icon.setImageBitmap((position & 1) == 1 ? mIcon1 : mIcon2);
	
	    return convertView;
	}
	
	static class ViewHolder {
	    TextView text;
	    ImageView icon;
	}
```
ViewHolder优雅写法
---
参考自base-adapter-helper [https://github.com/JoanZapata/base-adapter-helper]
```
public class ViewHolder {

    /** Views indexed with their IDs */
    private final SparseArray<View> views;

    private View convertView;

    protected ViewHolder(Context context, ViewGroup parent, int layoutId) {
        this.context = context;
        this.views = new SparseArray<View>();
        convertView = LayoutInflater.from(context).inflate(layoutId, parent, false);
        convertView.setTag(this);
    }

    public static ViewHolder get(Context context, View convertView, ViewGroup parent, int layoutId) {
        if (convertView == null) {
            return new ViewHolder(context, parent, layoutId);
        }
        return (ViewHolder) convertView.getTag();
    }

    /** Retrieve the convertView */
    public View getView() {
        return convertView;
    }

    @SuppressWarnings("unchecked")
    protected <T extends View> T retrieveView(int viewId) {
        View view = views.get(viewId);
        if (view == null) {
            view = convertView.findViewById(viewId);
            views.put(viewId, view);
        }
        return (T) view;
    }
}
```
调用时
```
	public View getView(int position, View convertView, ViewGroup parent) {

	    ViewHolder holder = ViewHolder.get(context, convertView, parent, layoutId);
		TextView text = holder.retrieveView(Id);
	    text..setText(DATA[position]);
	
	    return holder.getView（）;
	}
```


