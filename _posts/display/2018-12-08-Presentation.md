---
layout: post
title: "Presentation 演示"
date: 2018-12-08 21:36:00 +0800
categories: 分析与探索
tag: Display
---
* content
{:toc}

Markdown is a great mark language!

<!-- more -->

public class Presentation extends Dialog 
java.lang.Object  
   ↳android.app.Dialog  
    ↳android.app.Presentation

# Base class for presentations.演示类的基类

Presentation是一种特殊类型的对话框，其目的是在辅助显示中显示内容。Presentation在创建时与目标显示相关联，并根据显示的指标配置其上下文和资源配置。

A presentation is a special kind of dialog whose purpose is to present content on a secondary display. A Presentationis associated with the target Display at creation time and configures its context and resource configuration according to the display's metrics.

值得注意的是，Presentation的上下文与其包含的Activity上下文不同。重要的是要扩展Presentation的布局，并使用Presentation本身的上下文加载其他资源，以确保加载了目标屏幕的正确大小和密度的资源。

Notably, the Context of a presentation is different from the context of its containing Activity. It is important to inflate the layout of a presentation and load other resources using the presentation's own context to ensure that assets of the correct size and density for the target display are loaded.

当所附加的屏幕被移除时，Presentation将自动取消(请参见Dialog.cancel())。当Activity本身暂停或恢复时，Activity应该注意暂停和恢复在表示中播放的任何内容。

A presentation is automatically canceled (see Dialog.cancel()) when the display to which it is attached is removed. An activity should take care of pausing and resuming whatever content is playing within the presentation whenever the activity itself is paused or resumed.

## Choosing a presentation display选择一个演示屏幕

在展示一个Presentation之前，选择它将出现在哪个屏幕上是很重要的。选择一个演示屏幕有时是困难的，因为可能附加了多个屏幕。应用程序应该让系统选择合适的演示屏幕，而不是试图猜测哪种屏幕效果最好。

Before showing a Presentation it's important to choose the Display on which it will appear. Choosing a presentation display is sometimes difficult because there may be multiple displays attached. Rather than trying to guess which display is best, an application should let the system choose a suitable presentation display.

## There are two main ways to choose a Display.有两种主要的选择屏幕方法

### Using the media router to choose a presentation display使用媒体路由来选择演示屏幕
选择演示屏幕的最简单方法是使用MediaRouter API。媒体路由器服务跟踪系统上可用的音频和视频路由。当路由被选择或取消选择时，或者当路由的首选的演示屏幕发生变化时，媒体路由器发送通知。因此，应用程序可以简单地监视这些通知，并自动在首选的演示屏幕上显示或取消表示。

The easiest way to choose a presentation display is to use the MediaRouter API. The media router service keeps track of which audio and video routes are available on the system. The media router sends notifications whenever routes are selected or unselected or when the preferred presentation display of a route changes. So an application can simply watch for these notifications and show or dismiss a presentation on the preferred presentation display automatically.

如果它想在辅助显示上显示内容的话，首选的演示屏幕是媒体路由器建议应用程序应该使用的屏幕。有时可能没有首选的演示屏幕，在这种情况下，应用程序应该在本地显示其内容，而不使用演示。

The preferred presentation display is the display that the media router recommends that the application should use if it wants to show content on the secondary display. Sometimes there may not be a preferred presentation display in which case the application should show its content locally without using a presentation.

下面是如何使用媒体路由器使用MediaRouter.RouteInfo.getPresentationDisplay()在首选的表示显示器上创建和演示屏幕。

Here's how to use the media router to create and show a presentation on the preferred presentation display using MediaRouter.RouteInfo.getPresentationDisplay().
```
 MediaRouter mediaRouter = (MediaRouter) context.getSystemService(Context.MEDIA_ROUTER_SERVICE);
 MediaRouter.RouteInfo route = mediaRouter.getSelectedRoute();
 if (route != null) {
     Display presentationDisplay = route.getPresentationDisplay();
     if (presentationDisplay != null) {
         Presentation presentation = new MyPresentation(context, presentationDisplay);
         presentation.show();
     }
 }
```
下面来自ApiDemos的示例代码演示了如何使用媒体路由器在显示主Activity中的内容和显示可用的演示内容之间自动切换。

The following sample code from ApiDemos demonstrates how to use the media router to automatically switch between showing content in the main activity and showing the content in a presentation when a presentation display is available.
```
/**
 * <h3>Presentation Activity</h3>
 *
 * <p>
 * This demonstrates how to create an activity that shows some content
 * on a secondary display using a {@link Presentation}.
 * </p><p>
 * The activity uses the {@link MediaRouter} API to automatically detect when
 * a presentation display is available and to allow the user to control the
 * media routes using a menu item.  When a presentation display is available,
 * we stop showing content in the main activity and instead open up a
 * {@link Presentation} on the preferred presentation display.  When a presentation
 * display is removed, we revert to showing content in the main activity.
 * We also write information about displays and display-related events to
 * the Android log which you can read using <code>adb logcat</code>.
 * </p><p>
 * You can try this out using an HDMI or Wifi display or by using the
 * "Simulate secondary displays" feature in Development Settings to create a few
 * simulated secondary displays.  Each display will appear in the list along with a
 * checkbox to show a presentation on that display.
 * </p><p>
 * See also the {@link PresentationActivity} sample which
 * uses the low-level display manager to enumerate displays and to show multiple
 * simultaneous presentations on different displays.
 * </p>
 */
public class PresentationWithMediaRouterActivity extends Activity {
    private final String TAG = "PresentationWithMediaRouterActivity";

    private MediaRouter mMediaRouter;
    private DemoPresentation mPresentation;
    private GLSurfaceView mSurfaceView;
    private TextView mInfoTextView;
    private boolean mPaused;

    /**
     * Initialization of the Activity after it is first created.  Must at least
     * call {@link android.app.Activity#setContentView setContentView()} to
     * describe what is to be displayed in the screen.
     */
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // Be sure to call the super class.
        super.onCreate(savedInstanceState);

        // Get the media router service.
        mMediaRouter = (MediaRouter)getSystemService(Context.MEDIA_ROUTER_SERVICE);

        // See assets/res/any/layout/presentation_with_media_router_activity.xml for this
        // view layout definition, which is being set here as
        // the content of our screen.
        setContentView(R.layout.presentation_with_media_router_activity);

        // Set up the surface view for visual interest.
        mSurfaceView = (GLSurfaceView)findViewById(R.id.surface_view);
        mSurfaceView.setRenderer(new CubeRenderer(false));

        // Get a text view where we will show information about what's happening.
        mInfoTextView = (TextView)findViewById(R.id.info);
    }

    @Override
    protected void onResume() {
        // Be sure to call the super class.
        super.onResume();

        // Listen for changes to media routes.
        mMediaRouter.addCallback(MediaRouter.ROUTE_TYPE_LIVE_VIDEO, mMediaRouterCallback);

        // Update the presentation based on the currently selected route.
        mPaused = false;
        updatePresentation();
    }

    @Override
    protected void onPause() {
        // Be sure to call the super class.
        super.onPause();

        // Stop listening for changes to media routes.
        mMediaRouter.removeCallback(mMediaRouterCallback);

        // Pause rendering.
        mPaused = true;
        updateContents();
    }

    @Override
    protected void onStop() {
        // Be sure to call the super class.
        super.onStop();

        // Dismiss the presentation when the activity is not visible.
        if (mPresentation != null) {
            Log.i(TAG, "Dismissing presentation because the activity is no longer visible.");
            mPresentation.dismiss();
            mPresentation = null;
        }
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Be sure to call the super class.
        super.onCreateOptionsMenu(menu);

        // Inflate the menu and configure the media router action provider.
        getMenuInflater().inflate(R.menu.presentation_with_media_router_menu, menu);

        MenuItem mediaRouteMenuItem = menu.findItem(R.id.menu_media_route);
        MediaRouteActionProvider mediaRouteActionProvider =
                (MediaRouteActionProvider)mediaRouteMenuItem.getActionProvider();
        mediaRouteActionProvider.setRouteTypes(MediaRouter.ROUTE_TYPE_LIVE_VIDEO);

        // Return true to show the menu.
        return true;
    }

    private void updatePresentation() {
        // Get the current route and its presentation display.
        MediaRouter.RouteInfo route = mMediaRouter.getSelectedRoute(
                MediaRouter.ROUTE_TYPE_LIVE_VIDEO);
        Display presentationDisplay = route != null ? route.getPresentationDisplay() : null;

        // Dismiss the current presentation if the display has changed.
        if (mPresentation != null && mPresentation.getDisplay() != presentationDisplay) {
            Log.i(TAG, "Dismissing presentation because the current route no longer "
                    + "has a presentation display.");
            mPresentation.dismiss();
            mPresentation = null;
        }

        // Show a new presentation if needed.
        if (mPresentation == null && presentationDisplay != null) {
            Log.i(TAG, "Showing presentation on display: " + presentationDisplay);
            mPresentation = new DemoPresentation(this, presentationDisplay);
            mPresentation.setOnDismissListener(mOnDismissListener);
            try {
                mPresentation.show();
            } catch (WindowManager.InvalidDisplayException ex) {
                Log.w(TAG, "Couldn't show presentation!  Display was removed in "
                        + "the meantime.", ex);
                mPresentation = null;
            }
        }

        // Update the contents playing in this activity.
        updateContents();
    }

    private void updateContents() {
        // Show either the content in the main activity or the content in the presentation
        // along with some descriptive text about what is happening.
        if (mPresentation != null) {
            mInfoTextView.setText(getResources().getString(
                    R.string.presentation_with_media_router_now_playing_remotely,
                    mPresentation.getDisplay().getName()));
            mSurfaceView.setVisibility(View.INVISIBLE);
            mSurfaceView.onPause();
            if (mPaused) {
                mPresentation.getSurfaceView().onPause();
            } else {
                mPresentation.getSurfaceView().onResume();
            }
        } else {
            mInfoTextView.setText(getResources().getString(
                    R.string.presentation_with_media_router_now_playing_locally,
                    getWindowManager().getDefaultDisplay().getName()));
            mSurfaceView.setVisibility(View.VISIBLE);
            if (mPaused) {
                mSurfaceView.onPause();
            } else {
                mSurfaceView.onResume();
            }
        }
    }

    private final MediaRouter.SimpleCallback mMediaRouterCallback =
            new MediaRouter.SimpleCallback() {
        @Override
        public void onRouteSelected(MediaRouter router, int type, RouteInfo info) {
            Log.d(TAG, "onRouteSelected: type=" + type + ", info=" + info);
            updatePresentation();
        }

        @Override
        public void onRouteUnselected(MediaRouter router, int type, RouteInfo info) {
            Log.d(TAG, "onRouteUnselected: type=" + type + ", info=" + info);
            updatePresentation();
        }

        @Override
        public void onRoutePresentationDisplayChanged(MediaRouter router, RouteInfo info) {
            Log.d(TAG, "onRoutePresentationDisplayChanged: info=" + info);
            updatePresentation();
        }
    };

    /**
     * Listens for when presentations are dismissed.
     */
    private final DialogInterface.OnDismissListener mOnDismissListener =
            new DialogInterface.OnDismissListener() {
        @Override
        public void onDismiss(DialogInterface dialog) {
            if (dialog == mPresentation) {
                Log.i(TAG, "Presentation was dismissed.");
                mPresentation = null;
                updateContents();
            }
        }
    };

    /**
     * The presentation to show on the secondary display.
     * <p>
     * Note that this display may have different metrics from the display on which
     * the main activity is showing so we must be careful to use the presentation's
     * own {@link Context} whenever we load resources.
     * </p>
     */
    private final static class DemoPresentation extends Presentation {
        private GLSurfaceView mSurfaceView;

        public DemoPresentation(Context context, Display display) {
            super(context, display);
        }

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            // Be sure to call the super class.
            super.onCreate(savedInstanceState);

            // Get the resources for the context of the presentation.
            // Notice that we are getting the resources from the context of the presentation.
            Resources r = getContext().getResources();

            // Inflate the layout.
            setContentView(R.layout.presentation_with_media_router_content);

            // Set up the surface view for visual interest.
            mSurfaceView = (GLSurfaceView)findViewById(R.id.surface_view);
            mSurfaceView.setRenderer(new CubeRenderer(false));
        }

        public GLSurfaceView getSurfaceView() {
            return mSurfaceView;
        }
    }
}
```

### Using the display manager to choose a presentation display使用display manager 来选择一个演示屏幕

选择演示屏幕的另一种方法是直接使用DisplayManager API。display manager服务提供枚举和描述附加到系统上的所有屏幕的功能，包括可能用于演示的屏幕。

Another way to choose a presentation display is to use the DisplayManager API directly. The display manager service provides functions to enumerate and describe all displays that are attached to the system including displays that may be used for presentations.

display manager 跟踪系统中的所有显示。然而，并不是所有的显示器都适合显示演示文稿。例如，如果一个Activity试图在主屏幕上显示一个演示文稿，它可能会虚化它自己的内容(这就像在您的Activity上打开一个对话框)。（译注：弹窗后，背景通常是虚化的，以强化弹窗内容）

The display manager keeps track of all displays in the system. However, not all displays are appropriate for showing presentations. For example, if an activity attempted to show a presentation on the main display it might obscure its own content (it's like opening a dialog on top of your activity).

下面是如何用DisplayManager.getDisplays(String)和 DisplayManager.DISPLAY_CATEGORY_PRESENTATION类别来识别合适的演示文稿的屏幕。

Here's how to identify suitable displays for showing presentations using DisplayManager.getDisplays(String) and the DisplayManager.DISPLAY_CATEGORY_PRESENTATION category.
```
 DisplayManager displayManager = (DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);
 Display[] presentationDisplays = displayManager.getDisplays(DisplayManager.DISPLAY_CATEGORY_PRESENTATION);
 if (presentationDisplays.length > 0) {
     // If there is more than one suitable presentation display, then we could consider
     // giving the user a choice.  For this example, we simply choose the first display
     // which is the one the system recommends as the preferred presentation display.
     Display display = presentationDisplays[0];
     Presentation presentation = new MyPresentation(context, presentationDisplay);
     presentation.show();
 }
 ```
下面来自ApiDemos的示例代码演示了如何使用display manager 枚举屏幕和同时在多个演示屏幕上显示内容。

The following sample code from ApiDemos demonstrates how to use the display manager to enumerate displays and show content on multiple presentation displays simultaneously.
```
/**
 * <h3>Presentation Activity</h3>
 *
 * <p>
 * This demonstrates how to create an activity that shows some content
 * on a secondary display using a {@link Presentation}.
 * </p><p>
 * The activity uses the {@link DisplayManager} API to enumerate displays.
 * When the user selects a display, the activity opens a {@link Presentation}
 * on that display.  We show a different photograph in each presentation
 * on a unique background along with a label describing the display.
 * We also write information about displays and display-related events to
 * the Android log which you can read using <code>adb logcat</code>.
 * </p><p>
 * You can try this out using an HDMI or Wifi display or by using the
 * "Simulate secondary displays" feature in Development Settings to create a few
 * simulated secondary displays.  Each display will appear in the list along with a
 * checkbox to show a presentation on that display.
 * </p><p>
 * See also the {@link PresentationWithMediaRouterActivity} sample which
 * uses the media router to automatically select a secondary display
 * on which to show content based on the currently selected route.
 * </p>
 */
public class PresentationActivity extends Activity
        implements OnCheckedChangeListener, OnClickListener, OnItemSelectedListener {
    private final String TAG = "PresentationActivity";

    // Key for storing saved instance state.
    private static final String PRESENTATION_KEY = "presentation";

    // The content that we want to show on the presentation.
    private static final int[] PHOTOS = new int[] {
        R.drawable.frantic,
        R.drawable.photo1, R.drawable.photo2, R.drawable.photo3,
        R.drawable.photo4, R.drawable.photo5, R.drawable.photo6,
        R.drawable.sample_4,
    };

    private DisplayManager mDisplayManager;
    private DisplayListAdapter mDisplayListAdapter;
    private CheckBox mShowAllDisplaysCheckbox;
    private ListView mListView;
    private int mNextImageNumber;

    // List of presentation contents indexed by displayId.
    // This state persists so that we can restore the old presentation
    // contents when the activity is paused or resumed.
    private SparseArray<DemoPresentationContents> mSavedPresentationContents;

    // List of all currently visible presentations indexed by display id.
    private final SparseArray<DemoPresentation> mActivePresentations =
            new SparseArray<DemoPresentation>();

    /**
     * Initialization of the Activity after it is first created.  Must at least
     * call {@link android.app.Activity#setContentView setContentView()} to
     * describe what is to be displayed in the screen.
     */
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // Be sure to call the super class.
        super.onCreate(savedInstanceState);

        // Restore saved instance state.
        if (savedInstanceState != null) {
            mSavedPresentationContents =
                    savedInstanceState.getSparseParcelableArray(PRESENTATION_KEY);
        } else {
            mSavedPresentationContents = new SparseArray<DemoPresentationContents>();
        }

        // Get the display manager service.
        mDisplayManager = (DisplayManager)getSystemService(Context.DISPLAY_SERVICE);

        // See assets/res/any/layout/presentation_activity.xml for this
        // view layout definition, which is being set here as
        // the content of our screen.
        setContentView(R.layout.presentation_activity);

        // Set up checkbox to toggle between showing all displays or only presentation displays.
        mShowAllDisplaysCheckbox = (CheckBox)findViewById(R.id.show_all_displays);
        mShowAllDisplaysCheckbox.setOnCheckedChangeListener(this);

        // Set up the list of displays.
        mDisplayListAdapter = new DisplayListAdapter(this);
        mListView = (ListView)findViewById(R.id.display_list);
        mListView.setAdapter(mDisplayListAdapter);
    }

    @Override
    protected void onResume() {
        // Be sure to call the super class.
        super.onResume();

        // Update our list of displays on resume.
        mDisplayListAdapter.updateContents();

        // Restore presentations from before the activity was paused.
        final int numDisplays = mDisplayListAdapter.getCount();
        for (int i = 0; i < numDisplays; i++) {
            final Display display = mDisplayListAdapter.getItem(i);
            final DemoPresentationContents contents =
                    mSavedPresentationContents.get(display.getDisplayId());
            if (contents != null) {
                showPresentation(display, contents);
            }
        }
        mSavedPresentationContents.clear();

        // Register to receive events from the display manager.
        mDisplayManager.registerDisplayListener(mDisplayListener, null);
    }

    @Override
    protected void onPause() {
        // Be sure to call the super class.
        super.onPause();

        // Unregister from the display manager.
        mDisplayManager.unregisterDisplayListener(mDisplayListener);

        // Dismiss all of our presentations but remember their contents.
        Log.d(TAG, "Activity is being paused.  Dismissing all active presentation.");
        for (int i = 0; i < mActivePresentations.size(); i++) {
            DemoPresentation presentation = mActivePresentations.valueAt(i);
            int displayId = mActivePresentations.keyAt(i);
            mSavedPresentationContents.put(displayId, presentation.mContents);
            presentation.dismiss();
        }
        mActivePresentations.clear();
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        // Be sure to call the super class.
        super.onSaveInstanceState(outState);
        outState.putSparseParcelableArray(PRESENTATION_KEY, mSavedPresentationContents);
    }

    /**
     * Shows a {@link Presentation} on the specified display.
     */
    private void showPresentation(Display display, DemoPresentationContents contents) {
        final int displayId = display.getDisplayId();
        if (mActivePresentations.get(displayId) != null) {
            return;
        }

        Log.d(TAG, "Showing presentation photo #" + contents.photo
                + " on display #" + displayId + ".");

        DemoPresentation presentation = new DemoPresentation(this, display, contents);
        presentation.show();
        presentation.setOnDismissListener(mOnDismissListener);
        mActivePresentations.put(displayId, presentation);
    }

    /**
     * Hides a {@link Presentation} on the specified display.
     */
    private void hidePresentation(Display display) {
        final int displayId = display.getDisplayId();
        DemoPresentation presentation = mActivePresentations.get(displayId);
        if (presentation == null) {
            return;
        }

        Log.d(TAG, "Dismissing presentation on display #" + displayId + ".");

        presentation.dismiss();
        mActivePresentations.delete(displayId);
    }

    /**
     * Sets the display mode of the {@link Presentation} on the specified display
     * if it is already shown.
     */
    private void setPresentationDisplayMode(Display display, int displayModeId) {
        final int displayId = display.getDisplayId();
        DemoPresentation presentation = mActivePresentations.get(displayId);
        if (presentation == null) {
            return;
        }

        presentation.setPreferredDisplayMode(displayModeId);
    }

    private int getNextPhoto() {
        final int photo = mNextImageNumber;
        mNextImageNumber = (mNextImageNumber + 1) % PHOTOS.length;
        return photo;
    }

    /**
     * Called when the show all displays checkbox is toggled or when
     * an item in the list of displays is checked or unchecked.
     */
    @Override
    public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
        if (buttonView == mShowAllDisplaysCheckbox) {
            // Show all displays checkbox was toggled.
            mDisplayListAdapter.updateContents();
        } else {
            // Display item checkbox was toggled.
            final Display display = (Display)buttonView.getTag();
            if (isChecked) {
                DemoPresentationContents contents = new DemoPresentationContents(getNextPhoto());
                showPresentation(display, contents);
            } else {
                hidePresentation(display);
            }
            mDisplayListAdapter.updateContents();
        }
    }

    /**
     * Called when the Info button next to a display is clicked to show information
     * about the display.
     */
    @Override
    public void onClick(View v) {
        Context context = v.getContext();
        AlertDialog.Builder builder = new AlertDialog.Builder(context);
        final Display display = (Display)v.getTag();
        Resources r = context.getResources();
        AlertDialog alert = builder
                .setTitle(r.getString(
                        R.string.presentation_alert_info_text, display.getDisplayId()))
                .setMessage(display.toString())
                .setNeutralButton(R.string.presentation_alert_dismiss_text,
                        new DialogInterface.OnClickListener() {
                            @Override
                            public void onClick(DialogInterface dialog, int which) {
                                dialog.dismiss();
                            }
                    })
                .create();
        alert.show();
    }

    /**
     * Called when a display mode has been selected.
     */
    @Override
    public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
        final Display display = (Display)parent.getTag();
        final Display.Mode[] modes = display.getSupportedModes();
        setPresentationDisplayMode(display, position >= 1 && position <= modes.length ?
                modes[position - 1].getModeId() : 0);
    }

    /**
     * Called when a display mode has been unselected.
     */
    @Override
    public void onNothingSelected(AdapterView<?> parent) {
        final Display display = (Display)parent.getTag();
        setPresentationDisplayMode(display, 0);
    }

    /**
     * Listens for displays to be added, changed or removed.
     * We use it to update the list and show a new {@link Presentation} when a
     * display is connected.
     *
     * Note that we don't bother dismissing the {@link Presentation} when a
     * display is removed, although we could.  The presentation API takes care
     * of doing that automatically for us.
     */
    private final DisplayManager.DisplayListener mDisplayListener =
            new DisplayManager.DisplayListener() {
        @Override
        public void onDisplayAdded(int displayId) {
            Log.d(TAG, "Display #" + displayId + " added.");
            mDisplayListAdapter.updateContents();
        }

        @Override
        public void onDisplayChanged(int displayId) {
            Log.d(TAG, "Display #" + displayId + " changed.");
            mDisplayListAdapter.updateContents();
        }

        @Override
        public void onDisplayRemoved(int displayId) {
            Log.d(TAG, "Display #" + displayId + " removed.");
            mDisplayListAdapter.updateContents();
        }
    };

    /**
     * Listens for when presentations are dismissed.
     */
    private final DialogInterface.OnDismissListener mOnDismissListener =
            new DialogInterface.OnDismissListener() {
        @Override
        public void onDismiss(DialogInterface dialog) {
            DemoPresentation presentation = (DemoPresentation)dialog;
            int displayId = presentation.getDisplay().getDisplayId();
            Log.d(TAG, "Presentation on display #" + displayId + " was dismissed.");
            mActivePresentations.delete(displayId);
            mDisplayListAdapter.notifyDataSetChanged();
        }
    };

    /**
     * List adapter.
     * Shows information about all displays.
     */
    private final class DisplayListAdapter extends ArrayAdapter<Display> {
        final Context mContext;

        public DisplayListAdapter(Context context) {
            super(context, R.layout.presentation_list_item);
            mContext = context;
        }

        @Override
        public View getView(int position, View convertView, ViewGroup parent) {
            final View v;
            if (convertView == null) {
                v = ((Activity) mContext).getLayoutInflater().inflate(
                        R.layout.presentation_list_item, null);
            } else {
                v = convertView;
            }

            final Display display = getItem(position);
            final int displayId = display.getDisplayId();

            DemoPresentation presentation = mActivePresentations.get(displayId);
            DemoPresentationContents contents = presentation != null ?
                    presentation.mContents : null;
            if (contents == null) {
                contents = mSavedPresentationContents.get(displayId);
            }

            CheckBox cb = (CheckBox)v.findViewById(R.id.checkbox_presentation);
            cb.setTag(display);
            cb.setOnCheckedChangeListener(PresentationActivity.this);
            cb.setChecked(contents != null);

            TextView tv = (TextView)v.findViewById(R.id.display_id);
            tv.setText(v.getContext().getResources().getString(
                    R.string.presentation_display_id_text, displayId, display.getName()));

            Button b = (Button)v.findViewById(R.id.info);
            b.setTag(display);
            b.setOnClickListener(PresentationActivity.this);

            Spinner s = (Spinner)v.findViewById(R.id.modes);
            Display.Mode[] modes = display.getSupportedModes();
            if (contents == null || modes.length == 1) {
                s.setVisibility(View.GONE);
                s.setAdapter(null);
            } else {
                ArrayAdapter<String> modeAdapter = new ArrayAdapter<String>(mContext,
                        android.R.layout.simple_list_item_1);
                s.setVisibility(View.VISIBLE);
                s.setAdapter(modeAdapter);
                s.setTag(display);
                s.setOnItemSelectedListener(PresentationActivity.this);

                modeAdapter.add("<default mode>");

                for (Display.Mode mode : modes) {
                    modeAdapter.add(String.format("Mode %d: %dx%d/%.1ffps",
                            mode.getModeId(),
                            mode.getPhysicalWidth(), mode.getPhysicalHeight(),
                            mode.getRefreshRate()));
                    if (contents.displayModeId == mode.getModeId()) {
                        s.setSelection(modeAdapter.getCount() - 1);
                    }
                }
            }

            return v;
        }

        /**
         * Update the contents of the display list adapter to show
         * information about all current displays.
         */
        public void updateContents() {
            clear();

            String displayCategory = getDisplayCategory();
            Display[] displays = mDisplayManager.getDisplays(displayCategory);
            addAll(displays);

            Log.d(TAG, "There are currently " + displays.length + " displays connected.");
            for (Display display : displays) {
                Log.d(TAG, "  " + display);
            }
        }

        private String getDisplayCategory() {
            return mShowAllDisplaysCheckbox.isChecked() ? null :
                DisplayManager.DISPLAY_CATEGORY_PRESENTATION;
        }
    }

    /**
     * The presentation to show on the secondary display.
     *
     * Note that the presentation display may have different metrics from the display on which
     * the main activity is showing so we must be careful to use the presentation's
     * own {@link Context} whenever we load resources.
     */
    private final class DemoPresentation extends Presentation {

        final DemoPresentationContents mContents;

        public DemoPresentation(Context context, Display display,
                DemoPresentationContents contents) {
            super(context, display);
            mContents = contents;
        }

        /**
         * Sets the preferred display mode id for the presentation.
         */
        public void setPreferredDisplayMode(int modeId) {
            mContents.displayModeId = modeId;

            WindowManager.LayoutParams params = getWindow().getAttributes();
            params.preferredDisplayModeId = modeId;
            getWindow().setAttributes(params);
        }

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            // Be sure to call the super class.
            super.onCreate(savedInstanceState);

            // Get the resources for the context of the presentation.
            // Notice that we are getting the resources from the context of the presentation.
            Resources r = getContext().getResources();

            // Inflate the layout.
            setContentView(R.layout.presentation_content);

            final Display display = getDisplay();
            final int displayId = display.getDisplayId();
            final int photo = mContents.photo;

            // Show a caption to describe what's going on.
            TextView text = (TextView)findViewById(R.id.text);
            text.setText(r.getString(R.string.presentation_photo_text,
                    photo, displayId, display.getName()));

            // Show a n image for visual interest.
            ImageView image = (ImageView)findViewById(R.id.image);
            image.setImageDrawable(r.getDrawable(PHOTOS[photo]));

            GradientDrawable drawable = new GradientDrawable();
            drawable.setShape(GradientDrawable.RECTANGLE);
            drawable.setGradientType(GradientDrawable.RADIAL_GRADIENT);

            // Set the background to a random gradient.
            Point p = new Point();
            getDisplay().getSize(p);
            drawable.setGradientRadius(Math.max(p.x, p.y) / 2);
            drawable.setColors(mContents.colors);
            findViewById(android.R.id.content).setBackground(drawable);
        }
    }

    /**
     * Information about the content we want to show in the presentation.
     */
    private final static class DemoPresentationContents implements Parcelable {
        final int photo;
        final int[] colors;
        int displayModeId;

        public static final Creator<DemoPresentationContents> CREATOR =
                new Creator<DemoPresentationContents>() {
            @Override
            public DemoPresentationContents createFromParcel(Parcel in) {
                return new DemoPresentationContents(in);
            }

            @Override
            public DemoPresentationContents[] newArray(int size) {
                return new DemoPresentationContents[size];
            }
        };

        public DemoPresentationContents(int photo) {
            this.photo = photo;
            colors = new int[] {
                    ((int) (Math.random() * Integer.MAX_VALUE)) | 0xFF000000,
                    ((int) (Math.random() * Integer.MAX_VALUE)) | 0xFF000000 };
        }

        private DemoPresentationContents(Parcel in) {
            photo = in.readInt();
            colors = new int[] { in.readInt(), in.readInt() };
            displayModeId = in.readInt();
        }

        @Override
        public int describeContents() {
            return 0;
        }

        @Override
        public void writeToParcel(Parcel dest, int flags) {
            dest.writeInt(photo);
            dest.writeInt(colors[0]);
            dest.writeInt(colors[1]);
            dest.writeInt(displayModeId);
        }
    }
}
```

See also:

[for information on about live video routes and how to obtain the preferred presentation display for the current media route.](https://developer.android.com/reference/android/media/MediaRouter.html#ROUTE_TYPE_LIVE_VIDEO)

[for information on how to enumerate displays and receive notifications when displays are added or removed.](https://developer.android.com/reference/android/hardware/display/DisplayManager.html)
