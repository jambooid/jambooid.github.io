---
layout: post
title:  "activity performLaunchActivity 详解"
date:   2015-10-25 19:16:10
catalog:  true
tags:
    - android组件源码阅读




---

## 

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

​        ComponentName component = r.intent.getComponent();
​        if (component == null) {
​            component = r.intent.resolveActivity(
​                mInitialApplication.getPackageManager());
​            r.intent.setComponent(component);
​        }

​        if (r.activityInfo.targetActivity != null) {
​            component = new ComponentName(r.activityInfo.packageName,
​                    r.activityInfo.targetActivity);
​        }

​        ContextImpl appContext = createBaseContextForActivity(r);
​        Activity activity = null;
​        try {
​            java.lang.ClassLoader cl = appContext.getClassLoader();
​            activity = mInstrumentation.newActivity(
​                    cl, component.getClassName(), r.intent);
​            StrictMode.incrementExpectedActivityCount(activity.getClass());
​            r.intent.setExtrasClassLoader(cl);
​            r.intent.prepareToEnterProcess(isProtectedComponent(r.activityInfo),
​                    appContext.getAttributionSource());
​            if (r.state != null) {
​                r.state.setClassLoader(cl);
​            }
​        } catch (Exception e) {
​            if (!mInstrumentation.onException(activity, e)) {
​                throw new RuntimeException(
​                    "Unable to instantiate activity " + component

   + ": " + e.toString(), e);

     ​        }


​            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

​            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
​            if (localLOGV) Slog.v(
​                    TAG, r + ": app=" + app

   + ", appName=" + app.getPackageName()
     ackageInfo.getPackageName()
        + ", comp=" + r.intent.getComponent().toShortString()
          ackageInfo.getAppDir());

​            if (activity != null) {
​                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
​                Configuration config =
​                        new Configuration(mConfigurationController.getCompatConfiguration());
​                if (r.overrideConfig != null) {
​                    config.updateFrom(r.overrideConfig);
​                }
​                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "

   + r.activityInfo.name + " with config " + config);
     ;
                     if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                         window = r.mPendingRemoveWindow;
                         r.mPendingRemoveWindow = null;
                         r.mPendingRemoveWindowManager = null;
                     }

​                // Activity resources must be initialized with the same loaders as the
​                // application context.
​                appContext.getResources().addLoaders(
​                        app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

​                appContext.setOuterContext(activity);
​                activity.attach(appContext, this, getInstrumentation(), r.token,
​                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
​                        r.embeddedID, r.lastNonConfigurationInstances, config,
​                        r.referrer, r.voiceInteractor, window, r.configCallback,
​                        r.assistToken, r.shareableActivityToken);

​                if (customIntent != null) {
​                    activity.mIntent = customIntent;
​                }
​                r.lastNonConfigurationInstances = null;
​                checkAndBlockForNetworkAccess();
​                activity.mStartedActivity = false;
​                int theme = r.activityInfo.getThemeResource();
​                if (theme != 0) {
​                    activity.setTheme(theme);
​                }

​                if (r.mActivityOptions != null) {
​                    activity.mPendingOptions = r.mActivityOptions;
​                    r.mActivityOptions = null;
​                }
​                activity.mLaunchedFromBubble = r.mLaunchedFromBubble;
​                activity.mCalled = false;
​                if (r.isPersistable()) {
​                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
​                } else {
​                    mInstrumentation.callActivityOnCreate(activity, r.state);
​                }
​                if (!activity.mCalled) {
​                    throw new SuperNotCalledException(
​                        "Activity " + r.intent.getComponent().toShortString() +
​                        " did not call through to super.onCreate()");
​                }
​                r.activity = activity;
​                mLastReportedWindowingMode.put(activity.getActivityToken(),
​                        config.windowConfiguration.getWindowingMode());
​            }
​            r.setState(ON_CREATE);

​            // updatePendingActivityConfiguration() reads from mActivities to update
​            // ActivityClientRecord which runs in a different thread. Protect modifications to
​            // mActivities to avoid race.
​            synchronized (mResourcesManager) {
​                mActivities.put(r.token, r);
​            }        
​        return activity;
​    }
```



```sequence
H->H:handleMessage
note over H:如果是EXECUTE_TRANSACTION
H->TransactionExecutor:execute
TransactionExecutor->LaunchActityItem:execute
LaunchActityItem->ActivityThread:execute
ActivityThread->ActivityThread:handleLaunchActivity
ActivityThread->ActivityThread:performLaunchActivity
note over ActivityThread:用到了r.packageInfo
ActivityThread->ActivityThread:createBaseContextForActivity
ActivityThread->Instrumentation:callActivityOnCreate
```

ActivityThread是ClientTransactionHandler 的子类

`public `final class ActivityThread extends ClientTransactionHandler`
        implements ActivityThreadInternal {}`

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {   
        ActivityClientRecord r = client.getLaunchingActivity(token);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

ClientTransaction

```
   public void execute(ClientTransaction transaction) {
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

        final IBinder token = transaction.getActivityToken();
        if (token != null) {
            final Map<IBinder, ClientTransactionItem> activitiesToBeDestroyed =
                    mTransactionHandler.getActivitiesToBeDestroyed();
            final ClientTransactionItem destroyItem = activitiesToBeDestroyed.get(token);
            if (destroyItem != null) {
                if (transaction.getLifecycleStateRequest() == destroyItem) {
                    // It is going to execute the transaction that will destroy activity with the
                    // token, so the corresponding to-be-destroyed record can be removed.
                    activitiesToBeDestroyed.remove(token);
                }
                if (mTransactionHandler.getActivityClient(token) == null) {
                    // The activity has not been created but has been requested to destroy, so all
                    // transactions for the token are just like being cancelled.
                    Slog.w(TAG, tId(transaction) + "Skip pre-destroyed transaction:\n"
                            + transactionToString(transaction, mTransactionHandler));
                    return;
                }
            }
        }

        if (DEBUG_RESOLVER) Slog.d(TAG, transactionToString(transaction, mTransactionHandler));

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }
```

ActivityThread.H.handleMessage:

                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;