## 简介
用于Android客户端无埋点数据采集的Gradle插件。

Gradle Plugin for Codelessly Data Acquisition on Android Platform.

本插件基于开源项目[HiBeaver](https://github.com/BryanSharp/hibeaver)^_^

## 开发环境
- 语言：Groovy
- 字节码操作库：ASM5.0
- 工具：Android Studio 2.2
- Gradle：1.5+

## 使用

### 自定义参数

build.gradle中添加如下代码：

```
codelessdaConfig {
    //this will determine the name of this plugin transform, no practical use.
    pluginName = 'myPluginTest'
    //turn this on to make it print help content, default value is true
    showHelp = true
    //this flag will decide whether the log of the modifying process be printed or not, default value is false
    keepQuiet = false
    //this is a kit feature of the plugin, set it true to see the time consume of this build
    watchTimeConsume = false

    //this is the most important part, 3rd party JAR packages that want our plugin to inject;
    //our plugin will inject package defined in 'AndroidManifest.xml' and 'butterknife.internal.butterknife.internal.DebouncingOnClickListener' by default.
    //structure is like ['butterknife.internal','com.a.c'], type is HashSet<String>.
    //You can also specify the name of the class;
    //example: ['com.xxx.xxx.BaseFragment']
    targetPackages = []
}
```

## AOP在无埋点中的应用

### 1. 目标方法在Fragment中声明

目标方法：

- onResume()V
- onPause()V
- setUserVisibleHint(Z)V
- onHiddenChanged(Z)V

具体实现：

- 对app中指定包进行扫描，筛选出所有父类为`android/app/Fragment`或`android/support/v4/app/Fragment`的类。
- 对这些Fragment子类的`onResumed`，`onPaused`，`onHiddenChanged`，`setFragmentUserVisibleHint`方法的字节码进行修改，添加数据采集代码。

目标效果：

```
public class BaseFragment extends Fragment {
    public BaseFragment() {
    }

    public void onResume() {
        super.onResume();
        PluginAgent.onFragmentResume(this);
    }

    public void onHiddenChanged(boolean var1) {
        super.onHiddenChanged(var1);
        PluginAgent.onFragmentHiddenChanged(this);
    }

    public void onPause() {
        super.onPause();
        PluginAgent.onFragmentPause(this);
    }

    public void setUserVisibleHint(boolean var1) {
        super.setUserVisibleHint(var1);
        PluginAgent.setFragmentUserVisibleHint(this, var1);
    }
}
```

### 2. 目标方法在接口中声明

目标方法：

- onClick(Landroid/view/View;)V
- onClick(Landroid/content/DialogInterface;I)V
- onItemClick(Landroid/widget/AdapterView;Landroid/view/View;IJ)V
- onItemSelected(Landroid/widget/AdapterView;Landroid/view/View;IJ)V
- onGroupClick(Landroid/widget/ExpandableListView;Landroid/view/View;IJ)Z
- onChildClick(Landroid/widget/ExpandableListView;Landroid/view/View;IIJ)Z
- ...

具体实现：

- 对app中指定包进行扫描，筛选出实现了目标接口的类，在目标方法中添加数据采集代码。

> 例如，筛选出实现了`android/view/View$OnClickListener`接口的类，然后在`onClick(Landroid/view/View;)V`方法中注入采集数据的代码。

目标效果：

```
public class MainActivity extends Activity implements OnClickListener, android.content.DialogInterface.OnClickListener {
    public MainActivity() {
    }

    protected void onCreate(Bundle var1) {
        super.onCreate(var1);
        this.setContentView(2130968603);
        (new Builder(this)).setMessage("申请GPS权限").setPositiveButton("确定", new android.content.DialogInterface.OnClickListener() {
            public void onClick(DialogInterface var1, int var2) {
                PluginAgent.onClick(this, var1, var2);
            }
        }).show();
    }

    public void onClick(DialogInterface var1, int var2) {
        PluginAgent.onClick(this, var1, var2);
    }

    public void onClick(View var1) {
        PluginAgent.onClick(var1);
    }
}
```


## Groovy开发遇到的坑

- each闭包中的return相当于普通循环中的continue
- visitInsn尾部插入代码出错：at stack depth 0, expected type java.lang.Object but found int

```
for (def i = methodCell.paramsStart; i < methodCell.paramsStart + methodCell.paramsCount; i++) {  
	// Todo: 这里直接传入i居然报错：at stack depth 0, expected type java.lang.Object but found int；强制转换成Object才行。why?                                                                  
	methodVisitor.visitVarInsn(Opcodes.ALOAD, i);
}
```