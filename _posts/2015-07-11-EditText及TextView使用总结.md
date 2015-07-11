---
layout: default
title: Android EditText TextView使用总结
comments: true
---

在工作中遇到了EditText光标、复制粘贴、禁用等问题。这里做一些总结

###一、EditText的光标控制

```
setCursorVisible(boolean);

```
首先可使用上述方法控制光标是否显示,当然我们也可以使用

```
setFocusable(false);
setFocusableInTouchMode(false);

```


###二、EditText禁用复制粘贴

经调研，发现禁止复制粘贴需要禁止长按复制粘贴、禁止点击光标的蓝色指示复制粘贴、禁止双击出现复制粘贴。应该结合使用以下方式进行禁用[1][2]

```
edittext.setCustomSelectionActionModeCallback(new ActionMode.Callback() {

            public boolean onPrepareActionMode(ActionMode mode, Menu menu) {
                return false;
            }

            public void onDestroyActionMode(ActionMode mode) {                  
            }

            public boolean onCreateActionMode(ActionMode mode, Menu menu) {
                return false;
            }

            public boolean onActionItemClicked(ActionMode mode, MenuItem item) {
                return false;
            }
        });

edittext.setLongClick(false);

@Override
    public boolean isSuggestionsEnabled() {
        return false;
    }

/**
    * This is a replacement method for the base TextView class' method of the same name. This
    * method is used in hidden class android.widget.Editor to determine whether the PASTE/REPLACE popup
    * appears when triggered from the text insertion handle. Returning false forces this window
    * to never appear.
    * @return false
    */
    boolean canPaste() {
        return false;
    }

```

###三、如何禁止Edittext获得焦点或者禁止弹出系统软键盘？

当activity中包含edittext、并且没有设置activity软键盘属性android:windowSoftInputMode="stateAlwaysHidden|adjustPan时.进入activity会默认给该edittext获得焦点并且弹出系统软键盘，经调研实践，可通过clearFocus进行禁用获得焦点，同时需要注意，该activity存在其他focusable的view，才会让焦点从edittext转移到该view上。若只有一个可focus的edittext，那么clearFocus后仍然会重新让该edittext获得焦点

若需要获得焦点，不弹出系统软键盘，则可以使用如下方法[3]


```
setTextIsSelectable(true);

```
在[3]中，作者建议使用

```
editText.setRawInputType(InputType.TYPE_CLASS_TEXT);
setTextIsSelectable(true);

```
但经过实践，发现setRawInputType并没有用


###四、如何处理Edittext中内容与左边的空白问题？

```
setPadding(0, 0, 0, 0);

```


###五、如何禁用edittext和textview？

禁用edittext使用

```
setFocusable(false);
setFocusableInTouchMode(false);

```
而禁用textview

```
setEnable(false);

```

###六、如何去除TextView的文字上下的空格？
在开发中，遇到设定textviewtextSize，但是其仍然存在上下文字空格的问题，经调研，可设置如下方法

```
android:includeFontPadding="false"

```

同时，如果需要把文字和图片进行对齐，仅仅设置align等属性是不够的，因为文字还会存在上下空格的问题，我们可综合使用


```
<TextView
    android:layout_width="wrap_content"
    android:layout_height="14sp"
    android:textSize="@dimen/textsize_14sp"
    android:textColor="@color/text_black"
    android:includeFontPadding="false"/>
```


###七、如何获得textivew中内容的实际宽度？

这里需要结合rect 和paint来获得宽度 ,其中我们可以通过设置textSize来得到某一特定textSize的宽度。

```
private int getTextViewWidth() {
    Rect bounds = new Rect();
    Paint textPaint = this.getPaint();
    textPaint.setTextSize(textSize);
    String text = getText().toString();
    textPaint.getTextBounds(text, 0, text.length(), bounds);
    int height = bounds.height();
    int width = bounds.width();
    return width;
}

```


参考文献
[1]http://stackoverflow.com/questions/6275299/how-to-disable-copy-paste-from-to-edittext
[2]http://stackoverflow.com/questions/27869983/edittext-disable-paste-replace-menu-pop-up-on-text-selection-handler-click-even/28893714#28893714
[3]http://stackoverflow.com/questions/14282882/android-edit-text-cursor-doesnt-appear


