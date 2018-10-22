# Animator

本文主要分析Animator的源码

## 一个例子

```java
ValueAnimator anim = ValueAnimator.ofFloat(0f, 1f);
anim.setDuration(300);
anim.start();

```

## 源码追溯

```java
    public static ValueAnimator ofFloat(float... values) {
        ValueAnimator anim = new ValueAnimator();
        anim.setFloatValues(values);
        return anim;
    }
```
我们先看ValueAnimator的静态方法ofFloat，首先new一个ValueAnimator对象，再调用它的setFloatValues方法

```java
    public void setFloatValues(float... values) {
        if (values == null || values.length == 0) {
            return;
        }
        if (mValues == null || mValues.length == 0) {
            setValues(PropertyValuesHolder.ofFloat("", values));
        } else {
            PropertyValuesHolder valuesHolder = mValues[0];
            valuesHolder.setFloatValues(values);
        }
        // New property/values/target should cause re-initialization prior to starting
        mInitialized = false;
    }

    PropertyValuesHolder[] mValues;
```
将values传递到PropertyValuesHolder

```java
    public void setFloatValues(float... values) {
        mValueType = float.class;
        mKeyframes = KeyframeSet.ofFloat(values);
    }
```

赋值PropertyValuesHolder的成员mValueType，并将values传递到KeyframeSet

```java
    public static KeyframeSet ofFloat(float... values) {
        boolean badValue = false;
        int numKeyframes = values.length;
        FloatKeyframe keyframes[] = new FloatKeyframe[Math.max(numKeyframes,2)];
        if (numKeyframes == 1) {
            keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f);
            keyframes[1] = (FloatKeyframe) Keyframe.ofFloat(1f, values[0]);
            if (Float.isNaN(values[0])) {
                badValue = true;
            }
        } else {
            keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f, values[0]);
            for (int i = 1; i < numKeyframes; ++i) {
                keyframes[i] =
                        (FloatKeyframe) Keyframe.ofFloat((float) i / (numKeyframes - 1), values[i]);
                if (Float.isNaN(values[i])) {
                    badValue = true;
                }
            }
        }
        if (badValue) {
            Log.w("Animator", "Bad value (NaN) in float animator");
        }
        return new FloatKeyframeSet(keyframes);
    }

    public static Keyframe ofFloat(float fraction) {
        return new FloatKeyframe(fraction);
    }

    public static Keyframe ofFloat(float fraction, float value) {
        return new FloatKeyframe(fraction, value);
    }
```

我们看到最终返回一个FloatKeyframeSet，包含的每一个元素类型是FloatKeyframe，
每一个FloatKeyframe又包含了此关键帧的在整个动画过程中的位置（以百分数的形式），和它此时的value。

接下来setDuration不用说了

```java
    @Override
    public void start() {
        start(false);
    }

    private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        mReversing = playBackwards;
        mSelfPulse = !mSuppressSelfPulseRequested;
        // Special case: reversing from seek-to-0 should act as if not seeked at all.
        if (playBackwards && mSeekFraction != -1 && mSeekFraction != 0) {
            if (mRepeatCount == INFINITE) {
                // Calculate the fraction of the current iteration.
                float fraction = (float) (mSeekFraction - Math.floor(mSeekFraction));
                mSeekFraction = 1 - fraction;
            } else {
                mSeekFraction = 1 + mRepeatCount - mSeekFraction;
            }
        }
        mStarted = true;
        mPaused = false;
        mRunning = false;
        mAnimationEndRequested = false;
        // Resets mLastFrameTime when start() is called, so that if the animation was running,
        // calling start() would put the animation in the
        // started-but-not-yet-reached-the-first-frame phase.
        mLastFrameTime = -1;
        mFirstFrameTime = -1;
        mStartTime = -1;
        addAnimationCallback(0);

        if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
            // If there's no start delay, init the animation and notify start listeners right away
            // to be consistent with the previous behavior. Otherwise, postpone this until the first
            // frame after the start delay.
            startAnimation();
            if (mSeekFraction == -1) {
                // No seek, start at play time 0. Note that the reason we are not using fraction 0
                // is because for animations with 0 duration, we want to be consistent with pre-N
                // behavior: skip to the final value immediately.
                setCurrentPlayTime(0);
            } else {
                setCurrentFraction(mSeekFraction);
            }
        }
    }

```

首先设置一些状态，然后调用了addAnimationCallback

```java
    private void addAnimationCallback(long delay) {
        if (!mSelfPulse) {
            return;
        }
        getAnimationHandler().addAnimationFrameCallback(this, delay);
    }

    public AnimationHandler getAnimationHandler() {
        return AnimationHandler.getInstance();
    }
```
AnimationHandler是跟Looper类似的与线程绑定，同一个线程中get到的是同一个实例，这是通过
ThreadLocal实现的（ThreadLocalMap是以ThreadLocal为key，AnimationHandler的map）。

```java
    public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
        if (mAnimationCallbacks.size() == 0) {
            getProvider().postFrameCallback(mFrameCallback);
        }
        if (!mAnimationCallbacks.contains(callback)) {
            mAnimationCallbacks.add(callback);
        }

        if (delay > 0) {
            mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
        }
    }


```

ValueAnimator实现了AnimationFrameCallback接口，AnimationHandler的成员mAnimationCallbacks添加这个AnimationFrameCallback实例。

```java
    private AnimationFrameCallbackProvider getProvider() {
        if (mProvider == null) {
            mProvider = new MyFrameCallbackProvider();
        }
        return mProvider;
    }

    //MyFrameCallbackProvider.java
    final Choreographer mChoreographer = Choreographer.getInstance();
    public void postFrameCallback(Choreographer.FrameCallback callback) {               mChoreographer.postFrameCallback(callback);
    }

    //这里的callback
    //AnimationHandler.java
    private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            doAnimationFrame(getProvider().getFrameTime());
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    };
```

Choreographer实例同样是保存在ThreadLocal中。

```java
    //Choreographer.java
    public void postFrameCallback(FrameCallback callback) {
        postFrameCallbackDelayed(callback, 0);
    }
    public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
        if (callback == null) {
            throw new IllegalArgumentException("callback must not be null");
        }

        postCallbackDelayedInternal(CALLBACK_ANIMATION,
                callback, FRAME_CALLBACK_TOKEN, delayMillis);
    }

    private void postCallbackDelayedInternal(int callbackType,//== 1
            Object action, Object token, long delayMillis) {

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }

    private final CallbackQueue[] mCallbackQueues;

    //FrameHandler
    case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
```

将callback加入队列，如果没有延迟直接调用scheduleFrameLocked，如果有延迟通过FrameHandler定时发送消息触发doScheduleCallback，它最后还是调用scheduleFrameLocked

```java

    // Enable/disable vsync for animations and drawing.
    private static final boolean USE_VSYNC = SystemProperties.getBoolean(
            "debug.choreographer.vsync", true);

    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```

如果使用垂直同步，在Choreographer所在的Looper上执行

```java
    private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }

    //DisplayEventReceiver.java
        /**
     * Schedules a single vertical sync pulse to be delivered when the next
     * display frame begins.
     */
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }
```

调用到了native层暂时不再继续往下深入。

如果没有使用垂直同步，会调用doFrame

```java
    //Choreographer.java
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }

            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            final long jitterNanos = startNanos - frameTimeNanos;
            //mFrameIntervalNanos代表每一帧的纳秒数
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                //计算需要跳过的帧数
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;

                frameTimeNanos = startNanos - lastFrameOffset;
            }

            if (frameTimeNanos < mLastFrameTimeNanos) {

                scheduleVsyncLocked();//还是调用这个方法，会调用到native层
                return;
            }

            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled = false;
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);//锁定状态

            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
            //之前队列中add的Callback的类型是CALLBACK_ANIMATION
            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();//解锁状态
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

    }
```

```java
    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            // We use "now" to determine when callbacks become due because it's possible
            // for earlier processing phases in a frame to post callbacks that should run
            // in a following phase, such as an input event that causes an animation to start.
            final long now = System.nanoTime();
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);//从队列中取出dueTime不大于now的callback
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;

            // Update the frame time if necessary when committing the frame.
            // We only update the frame time if we are more than 2 frames late reaching
            // the commit phase.  This ensures that the frame time which is observed by the
            // callbacks will always increase from one frame to the next and never repeat.
            // We never want the next frame's starting frame time to end up being less than
            // or equal to the previous frame's commit frame time.  Keep in mind that the
            // next frame has most likely already been scheduled by now so we play it
            // safe by ensuring the commit time is always at least one frame behind.
            if (callbackType == Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos = now - frameTimeNanos;
                Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                            + mFrameIntervalNanos;
                    if (DEBUG_JANK) {
                        Log.d(TAG, "Commit callback delayed by " + (jitterNanos * 0.000001f)
                                + " ms which is more than twice the frame interval of "
                                + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                                + "Setting frame time to " + (lastFrameOffset * 0.000001f)
                                + " ms in the past.");
                        mDebugPrintNextFrameTimeDelta = true;
                    }
                    frameTimeNanos = now - lastFrameOffset;
                    mLastFrameTimeNanos = frameTimeNanos;
                }
            }
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                            + ", action=" + c.action + ", token=" + c.token
                            + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
                }
                c.run(frameTimeNanos);//执行callback的run方法，这里的callback实际是CallbackRecord，它的成员action代表FrameCallback
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

    //CallbackRecord.class
    public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
```

回调了doFrame方法，因为token是FRAME_CALLBACK_TOKEN。

```java
    private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            doAnimationFrame(getProvider().getFrameTime());
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    };
```

doFrame是在Choreographer所在的Looper线程中执行的。

```java
    private void doAnimationFrame(long frameTime) {
        long currentTime = SystemClock.uptimeMillis();
        final int size = mAnimationCallbacks.size();
        for (int i = 0; i < size; i++) {
            final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
            if (callback == null) {
                continue;
            }
            if (isCallbackDue(callback, currentTime)) {//如果不延迟
                callback.doAnimationFrame(frameTime);//这里的callback就是ValueAnimator
                if (mCommitCallbacks.contains(callback)) {
                    getProvider().postCommitCallback(new Runnable() {
                        @Override
                        public void run() {
                            commitAnimationFrame(callback, getProvider().getFrameTime());
                        }
                    });
                }
            }
        }
        cleanUpList();
    }
```

调用到ValueAnimator的doAnimationFrame，是动画起作用的主要逻辑，会看AnimationHandler的mFrameCallback如果mAnimationCallbacks.size() > 0会再调用MyFrameCallbackProvider的
postFrameCallback，这和一开始是一样的会在下一帧时间调用到ValueAnimator的doAnimationFrame，直到ValueAniator的endAnimation被调用，callback从AnimationHandler的mAnimationCallbacks中移除。

```java
    public final boolean doAnimationFrame(long frameTime) {
        if (mStartTime < 0) {
            // First frame. If there is start delay, start delay count down will happen *after* this
            // frame.
            mStartTime = mReversing ? frameTime : frameTime + (long) (mStartDelay * sDurationScale);
        }

        // Handle pause/resume
        if (mPaused) {
            mPauseTime = frameTime;
            removeAnimationCallback();
            return false;
        } else if (mResumed) {
            mResumed = false;
            if (mPauseTime > 0) {
                // Offset by the duration that the animation was paused
                mStartTime += (frameTime - mPauseTime);
            }
        }

        if (!mRunning) {
            // If not running, that means the animation is in the start delay phase of a forward
            // running animation. In the case of reversing, we want to run start delay in the end.
            if (mStartTime > frameTime && mSeekFraction == -1) {
                // This is when no seek fraction is set during start delay. If developers change the
                // seek fraction during the delay, animation will start from the seeked position
                // right away.
                return false;
            } else {
                // If mRunning is not set by now, that means non-zero start delay,
                // no seeking, not reversing. At this point, start delay has passed.
                mRunning = true;
                startAnimation();
            }
        }

        if (mLastFrameTime < 0) {
            if (mSeekFraction >= 0) {
                long seekTime = (long) (getScaledDuration() * mSeekFraction);
                mStartTime = frameTime - seekTime;
                mSeekFraction = -1;
            }
            mStartTimeCommitted = false; // allow start time to be compensated for jank
        }
        mLastFrameTime = frameTime;
        // The frame time might be before the start time during the first frame of
        // an animation.  The "current time" must always be on or after the start
        // time to avoid animating frames at negative time intervals.  In practice, this
        // is very rare and only happens when seeking backwards.
        final long currentTime = Math.max(frameTime, mStartTime);
        boolean finished = animateBasedOnTime(currentTime);

        if (finished) {
            endAnimation();
        }
        return finished;
    }

```

关键方法是animateBasedOnTime

```java
    boolean animateBasedOnTime(long currentTime) {
        boolean done = false;
        if (mRunning) {
            final long scaledDuration = getScaledDuration();
            final float fraction = scaledDuration > 0 ?
                    (float)(currentTime - mStartTime) / scaledDuration : 1f;
            final float lastFraction = mOverallFraction;
            final boolean newIteration = (int) fraction > (int) lastFraction;
            final boolean lastIterationFinished = (fraction >= mRepeatCount + 1) &&
                    (mRepeatCount != INFINITE);
            if (scaledDuration == 0) {
                // 0 duration animator, ignore the repeat count and skip to the end
                done = true;
            } else if (newIteration && !lastIterationFinished) {
                // Time to repeat
                if (mListeners != null) {
                    int numListeners = mListeners.size();
                    for (int i = 0; i < numListeners; ++i) {
                        mListeners.get(i).onAnimationRepeat(this);
                    }
                }
            } else if (lastIterationFinished) {
                done = true;
            }
            mOverallFraction = clampFraction(fraction);
            float currentIterationFraction = getCurrentIterationFraction(
                    mOverallFraction, mReversing);
            animateValue(currentIterationFraction);
        }
        return done;
    }

```

又调用到了animateValue

```java
    void animateValue(float fraction) {
        fraction = mInterpolator.getInterpolation(fraction);//插值器换算百分百
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].calculateValue(fraction);//PropertyValueHolder通过百分百计算value
        }
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }

```

```java
    void calculateValue(float fraction) {
        Object value = mKeyframes.getValue(fraction);
        mAnimatedValue = mConverter == null ? value : mConverter.convert(value);
    }
    //FloatKeyframeSet.java
    @Override
    public Object getValue(float fraction) {
        return getFloatValue(fraction);
    }

    //FloatEvaluator
    public Float evaluate(float fraction, Number startValue, Number endValue) {
        float startFloat = startValue.floatValue();
        return startFloat + fraction * (endValue.floatValue() - startFloat);
    }
```

getValue大体就是通过百分百、起点和终点来计算的。
计算后的值或者通过Converter转换或者直接使用。

ValueAnimator每一帧计算一次value就形成了动画，如果是ObjectAnimator还有一个target

animateValue会多调用setAnimatedValue

```java
    void setAnimatedValue(Object target) {
        if (mProperty != null) {//这里一般==null
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {//通过反射调用target的setter方法设置属性
            try {
                mTmpValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
            } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
            }
        }
    }
```

## 小结

属性动画通过多个关键值形成关键帧，start方法会触发在编舞者所在的Looper每一帧回调一次doAnimationFrame,会通过百分百计算value，也可能需要设置value给target。
所以属性动画实际上在动画过程中真正改变了target的属性值，这一点和View动画的只是通过Matrix变换显示效果是有本质区别的。