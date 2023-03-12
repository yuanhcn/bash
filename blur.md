### 窗口模糊

在 Android 12 中，公共 API 可用于实现窗口模糊效果，例如背景模糊和背后模糊。窗口模糊或交叉窗口模糊用于模糊给定窗口后面的屏幕。窗口模糊有两种类型，可以用来实现不同的视觉效果：

* 背景模糊允许您创建背景模糊的窗口，从而创建磨砂玻璃效果。
* Blur behind允许您模糊（对话框）窗口后面的整个屏幕，创建景深效果。


窗口模糊功能适用于整个窗口，这意味着当您的窗口后面有另一个应用程序时它也可以工作。此效果与模糊渲染效果不同，后者会模糊同一窗口内的内容。窗口模糊对于对话框、底部工作表和其他浮动窗口很有用。
官方文档中，两种效果单独使用和组合使用的效果，如下图所示：

* 仅窗口背景模糊
![51c5c1f1fb2ca190a0d1a52b2e0822db.png](en-resource://database/685:1)
* 仅窗口后面模糊
![72799ee76e5470fa23737a2477318b2f.png](en-resource://database/687:1)
* 窗口背景和后面都模糊
![bbc4db6a584b743dbb79483591a2bb7c.png](en-resource://database/689:1)

#### 使用方法

##### 设备制造商

要检查您的设备是否支持窗口模糊，请执行以下操作：

* 确保设备可以处理额外的 GPU 负载。低端设备可能无法处理额外的负载，这可能会导致丢帧。仅在具有足够 GPU 能力的测试设备上启用窗口模糊。
* 如果您有自定义渲染引擎，请确保您的渲染引擎实现了模糊逻辑。默认的 Android 12渲染引擎在BlurFilter.cpp中实现模糊逻辑。


设备是否可以支持窗口模糊的系统属性
```
PRODUCT_VENDOR_PROPERTIES += \       ro.surface_flinger.supports_background_blur=1
```

###### 打开和关闭窗口模糊的方法

要测试窗口 UI 如何使用窗口模糊效果呈现，请使用以下方法之一启用或禁用模糊：

* 从开发者选项：
设置 -> 系统 -> 开发者选项 -> 硬件加速渲染 -> 允许窗口级模糊

* Root下终端命令：
```
adb shell wm disable-blur 1 # 1 disables window blurs, 0 allows them
```

##### 应用开发者

应用程序开发人员必须提供模糊半径来创建模糊效果。模糊半径控制模糊的密集程度，即半径越高，模糊越密集。 0px的模糊意味着没有模糊。对于后面的模糊，20px 的半径可产生良好的景深效果，而80px的背景模糊半径可产生良好的磨砂玻璃效果。避免高于150px的模糊半径，因为这会显着影响性能。
要获得所需的模糊效果并提高可读性，请选择一个模糊半径值并辅以半透明的颜色层。
###### 背景模糊

在浮动窗口上使用背景模糊来创建窗口背景效果，这是底层内容的模糊图像。要为窗口添加模糊背景，请执行以下操作：

1. 调用Window#setBackgroundBlurRadius(int)设置背景模糊半径。或者，在窗口主题中，设置R.attr.windowBackgroundBlurRadius 。
```
getWindow().setBackgroundBlurRadius(mBackgroundBlurRadius);

```

2. 将R.attr.windowIsTranslucent设置为 true 以使窗口半透明。模糊是在窗口表面下绘制的，因此窗口需要半透明才能让模糊可见。

```
<style name="Theme.xxx" >    
    <item name="android:windowIsTranslucent">true</item>
</style>

```

3. 可选地，调用Window#setBackgroundDrawableResource(int)以添加具有半透明颜色的矩形窗口背景。或者，在窗口主题中，设置R.attr.windowBackground 。


4. 对于带有圆角的窗口，通过将带有圆角的ShapeDrawable设置为窗口背景可绘制对象来确定模糊区域的圆角。

5. 监听模糊启用和禁用状态，在启用和禁用时分别处理。

```
getWindowManager().addCrossWindowBlurEnabledListener( windowBlurEnabledListener);
```

###### 背后模糊

后面的模糊模糊了窗户后面的整个屏幕。此效果用于通过模糊窗口后面屏幕上的任何内容来将用户的注意力引导到窗口内容。

要模糊窗口后面的内容，请执行以下步骤：
1. 将FLAG_BLUR_BEHIND添加到窗口标志，以启用后面的模糊。或者，在窗口主题中，设置R.attr.windowBlurBehindEnabled 。
```
getWindow().addFlags(WindowManager.LayoutParams.FLAG_BLUR_BEHIND);

```
2. 调用WindowManager.LayoutParams#setBlurBehindRadius设置半径后面的模糊。或者，在窗口主题中，设置R.attr.windowBlurBehindRadius 。
```
private final int mBlurBehindRadius = 20;

……

getWindow().getAttributes().setBlurBehindRadius(mBlurBehindRadius);

```
3. 选择一个补充的暗淡量,由于覆盖原始的暗淡量，使之和模糊背景适配。
```
getWindow().setDimAmount（xxx）
```

4. 监听模糊启用和禁用状态，在启用和禁用时分别处理。

```
getWindowManager().addCrossWindowBlurEnabledListener( windowBlurEnabledListener);

```

### View背景模糊

原生API仅提供了窗口级别的模糊方案，通过分析原生API实现背景高斯模糊效果底层逻辑，也能实现View级别的背景的模糊。
#### Android12中和高斯模糊相关的关键代码和属性样式

* 关键代码：

```
# 背景高斯模糊
android.view.Window#setBackgroundBlurRadius(int blurRadius)

# 后方屏幕高斯模糊
android.view.WindowManager.LayoutParams#FLAG_BLUR_BEHIND
android.view.WindowManager.LayoutParams#setBlurBehindRadius(int blurBehindRadius)

# 跨窗口高斯模糊监听器
android.view.WindowManager#isCrossWindowBlurEnabled()
android.view.WindowManager#addCrossWindowBlurEnabledListener(@NonNull Consumer<Boolean> listener)
android.view.WindowManager#removeCrossWindowBlurEnabledListener(Consumer<Boolean> listener)

```

* 关键属性样式：
```
<declare-styleable name="Theme">   
   <attr name="windowBackgroundBlurRadius" format="dimension" />
   <attr name="windowBlurBehindRadius" format="dimension"/>
   <attr name="windowBlurBehindEnabled" format="boolean" />
</declare-styleable>

```

#### 背景高斯模糊API源码分析

1. 背景高斯模糊所对应的API是抽象类Window的setBackgroundBlurRadius方法，在Android中Window只有一个实现类PhoneWindow，PhoneWindow类中的setBackgroundBlurRadius方法如下所示：

```
public class PhoneWindow extends Window implements MenuBuilder.Callback {
    @Override
    public final void setBackgroundBlurRadius(int blurRadius) {
        super.setBackgroundBlurRadius(blurRadius);
        if (CrossWindowBlurListeners.CROSS_WINDOW_BLUR_SUPPORTED) {
            if (mBackgroundBlurRadius != Math.max(blurRadius, 0)) {
                mBackgroundBlurRadius = Math.max(blurRadius, 0);
                mDecor.setBackgroundBlurRadius(mBackgroundBlurRadius);
            }
        }
    }
}
```

setBackgroundBlurRadius首先会判断是否支持窗口模糊，然后判断用户有没有设置blurRadius，如果设置了会继续调用DecorView的setBackgroundBlurRadius方法。


2. DecorView的setBackgroundBlurRadius方法如下所示：

```
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {

    void setBackgroundBlurRadius(int blurRadius) {
        mOriginalBackgroundBlurRadius = blurRadius;
        if (blurRadius > 0) {
            if (mCrossWindowBlurEnabledListener == null) {
                mCrossWindowBlurEnabledListener = enabled -> {
                    mCrossWindowBlurEnabled = enabled;
                    //更新背景高斯模糊效果的半径
                    updateBackgroundBlurRadius();
                };
                getContext().getSystemService(WindowManager.class)
                        .addCrossWindowBlurEnabledListener(mCrossWindowBlurEnabledListener);
                getViewTreeObserver().addOnPreDrawListener(mBackgroundBlurOnPreDrawListener);
            } else {
                //更新背景高斯模糊效果的半径
                updateBackgroundBlurRadius();
            }
        } else if (mCrossWindowBlurEnabledListener != null) {
            //更新背景高斯模糊效果的半径
            updateBackgroundBlurRadius();
            removeBackgroundBlurDrawable();
        }
    }
    
 }
```

3. DecorView的setBackgroundBlurRadius方法根据不同的情况会执行不同的分支，但最终都会调用一个关键的方法updateBackgroundBlurRadius。

```
    //更新背景高斯模糊效果半径
    private void updateBackgroundBlurRadius() {
        //如果viewRootImpl为空直接返回
        if (getViewRootImpl() == null) return;

        //获取背景高斯模糊效果半径
        mBackgroundBlurRadius = mCrossWindowBlurEnabled && mWindow.isTranslucent()
                ? mOriginalBackgroundBlurRadius : 0;

        if (mBackgroundBlurDrawable == null && mBackgroundBlurRadius > 0) {
            //调用ViewRootImpl的方法创建高斯模糊Drawable对象
            mBackgroundBlurDrawable = getViewRootImpl().createBackgroundBlurDrawable();
            updateBackgroundDrawable();
        }

        if (mBackgroundBlurDrawable != null) {
            mBackgroundBlurDrawable.setBlurRadius(mBackgroundBlurRadius);
        }
    }

```

updateBackgroundBlurRadius方法首先获取ViewRootImpl实例对象，如果为空直接返回；如果不为空则获取背景高斯模糊效果半径，当mCrossWindowBlurEnabled 为true且窗口是透明样式的时候，才会获取之前设置的高斯模糊效果半径，否则高斯模糊效果半径直接设置为0；然后会检测mBackgroundBlurDrawable是否为空，如果为空且获取的mBackgroundBlurRadius大于0，便会调用ViewRootImpl的createBackgroundBlurDrawable方法创建BackgroundBlurDrawable对象实例，BackgroundBlurDrawable对象是系统实现背景高斯模糊效果的关键。

4. ViewRootImpl的createBackgroundBlurDrawable方法如下所示：

```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks,
        AttachedSurfaceControl {
        
	private final BackgroundBlurDrawable.Aggregator mBlurRegionAggregator =
            new BackgroundBlurDrawable.Aggregator(this);

    public BackgroundBlurDrawable createBackgroundBlurDrawable() {
        return mBlurRegionAggregator.createBackgroundBlurDrawable(mContext);
    }
 }

```

createBackgroundBlurDrawable方法仅仅是进一步调用mBlurRegionAggregator的createBackgroundBlurDrawable方法，mBlurRegionAggregator是BackgroundBlurDrawable的一个内部静态类。


5. 重新回到第3步DecorView的updateBackgroundBlurRadius方法中：

```
    //更新高斯模糊背景的高斯半径
    private void updateBackgroundBlurRadius() {
    	...代码省略...
        if (mBackgroundBlurDrawable == null && mBackgroundBlurRadius > 0) {
            mBackgroundBlurDrawable = getViewRootImpl().createBackgroundBlurDrawable();
            //更新背景Drawable
            updateBackgroundDrawable();
        }

        if (mBackgroundBlurDrawable != null) {
            mBackgroundBlurDrawable.setBlurRadius(mBackgroundBlurRadius);
        }
    }

```

在调用createBackgroundBlurDrawable方法为mBackgroundBlurDrawable进行赋值之后，会继续调用DecorView的updateBackgroundDrawable方法来更新当前DecorView所对应的背景Drawable，最后再使用mBackgroundBlurRadius更新mBackgroundBlurDrawable的模糊圆角半径数值。


6. DecorView的updateBackgroundDrawable方法如下所示：
```
    private void updateBackgroundDrawable() {
        // Background insets can be null if super constructor calls setBackgroundDrawable.
        if (mBackgroundInsets == null) {
            mBackgroundInsets = Insets.NONE;
        }

        if (mBackgroundInsets.equals(mLastBackgroundInsets)
                && mBackgroundBlurDrawable == mLastBackgroundBlurDrawable
                && mLastOriginalBackgroundDrawable == mOriginalBackgroundDrawable) {
            return;
        }

        Drawable destDrawable = mOriginalBackgroundDrawable;
        if (mBackgroundBlurDrawable != null) {
            //将刚刚创建的类型为BackgroundBlurDrawable的mBackgroundBlurDrawable
            //和原来类型为Drawable的mOriginalBackgroundDrawable合并成一个LayerDrawable实例对象
            destDrawable = new LayerDrawable(new Drawable[]{mBackgroundBlurDrawable,
                    mOriginalBackgroundDrawable});
        }

        if (destDrawable != null && !mBackgroundInsets.equals(Insets.NONE)) {
            //将当前类型为LayerDrawable的destDrawable再封装成InsetDrawable实例对象
            destDrawable = new InsetDrawable(destDrawable,
                    mBackgroundInsets.left, mBackgroundInsets.top,
                    mBackgroundInsets.right, mBackgroundInsets.bottom) {

                /**
                 * Return inner padding so we don't apply the padding again in
                 * {@link DecorView#drawableChanged()}
                 */
                @Override
                public boolean getPadding(Rect padding) {
                    return getDrawable().getPadding(padding);
                }
            };
        }
        //调用父类方法设置这个类的背景Drawable
        super.setBackgroundDrawable(destDrawable);

        mLastBackgroundInsets = mBackgroundInsets;
        mLastBackgroundBlurDrawable = mBackgroundBlurDrawable;
        mLastOriginalBackgroundDrawable = mOriginalBackgroundDrawable;
    }

```

updateBackgroundDrawable方法中最关键的一点就是将前面通过ViewImpl创建的类型为BackgroundBlurDrawable的高斯模糊mBackgroundBlurDrawable对象和原来类型为Drawable的原始背景mOriginalBackgroundDrawable对象合并成一个LayerDrawable实例对象，最后再调用父类（View）的setBackgroundDrawable将LayerDrawable设置为新的背景。

#### View背景模糊方案


结合前面对PhoneWindow类中的setBackgroundBlurRadius方法的分析可以知道，该API所作的主要工作主要包含两个方面：

1. 调用ViewImpl的createBackgroundBlurDrawable方法创建BackgroundBlurDrawable实例对象。
2. 将BackgroundBlurDrawable和原本的背景Drawable文件合并成一个全新的LayerDrawable实例对象，最后再将LayerDrawable实例对象设置成DecorView的背景，这样就实现了Decorview的背景高斯模糊效果。
3. 那么，我们可不可以将本来设置DecorView的LayerDrawable实例对象设置成其他view的背景，以实现view的背景模糊？答案是，可以。


获取BackgroundBlurDrawable实例对象的方法：

* 系统开发者，可以使用隐藏API：

```
    /**
     * 通过添加framework.jar依赖获取BackgroundBlurDrawable实例对象
     *
     * @param target
     * @return
     */
    private BackgroundBlurDrawable getBackgroundBlurDrawableByFramework(View target) {
        BackgroundBlurDrawable backgroundBlurDrawable = target.
getViewRootImpl().createBackgroundBlurDrawable();
        backgroundBlurDrawable.setBlurRadius(mBackgroundBlurRadius);
        backgroundBlurDrawable.setCornerRadius(mBackgroundCornersRadius);
        return backgroundBlurDrawable;
    }


```

* 应用开发这也可以使用反射：
```
/**
 * 通过反射获取BackgroundBlurDrawable对象实例
 *
 * @param target
 * @return
 */
public Drawable getBackgroundBlurDrawableByReflect(View target ,int mBackgroundBlurRadius, int mBackgroundCornersRadius) {
    Drawable drawable = null;
    try {
        // 反射获取ViewRootImpl对象
        Object viewRootImpl = null;
        Method getViewRootImpl =  target.getClass().getDeclaredMethod("getViewRootImpl");
        viewRootImpl = getViewRootImpl.invoke(target);

        //调用ViewRootImpl的createBackgroundBlurDrawable方法创建实例
        Method method_createBackgroundBlurDrawable = viewRootImpl.getClass().getDeclaredMethod("createBackgroundBlurDrawable");
        method_createBackgroundBlurDrawable.setAccessible(true);
        drawable = (Drawable) method_createBackgroundBlurDrawable.invoke(viewRootImpl);
        //调用BackgroundBlurDrawable的setBlurRadius方法
        Method method_setBlurRadius = drawable.getClass().getDeclaredMethod("setBlurRadius", int.class);
        method_setBlurRadius.setAccessible(true);
        method_setBlurRadius.invoke(drawable, mBackgroundBlurRadius);
        //调用BackgroundBlurDrawable的setCornerRadius方法
        Method method_setCornerRadius = drawable.getClass().getDeclaredMethod("setCornerRadius", int.class);
        method_setCornerRadius.setAccessible(true);
        method_setCornerRadius.invoke(drawable, mBackgroundCornersRadius);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return drawable;
}

```

