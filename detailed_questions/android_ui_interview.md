# Android UI / View System Interview Guide

A comprehensive interview-prep guide for the **classic Android UI / View system** — `View`, `ViewGroup`, layouts, custom views, the `Canvas`, `RecyclerView`, dialogs, toasts, and text/theming. This file deliberately covers the **imperative XML + `View` toolkit**. Jetpack Compose (the modern declarative toolkit) is covered in a separate file.

> Note on currency (2026): The `View` system remains fully supported and is still the foundation of millions of production apps and of Compose interop. However, **`ListView` and `RelativeLayout` are effectively legacy** — for new code prefer `RecyclerView` over `ListView`, `ConstraintLayout` over `RelativeLayout`/deeply nested `LinearLayout`, and consider **Jetpack Compose** for greenfield UI. Each relevant question notes the modern alternative.

---

## Table of Contents

### Views and ViewGroups
1. [What is a `View` in Android?](#1-what-is-a-view-in-android)
2. [What are `ViewGroup`s and how do they differ from `View`s?](#2-what-are-viewgroups-and-how-do-they-differ-from-views)
3. [Difference between `View.GONE` and `View.INVISIBLE`](#3-difference-between-viewgone-and-viewinvisible)
4. [The View lifecycle and how to create a custom `View`](#4-the-view-lifecycle-and-how-to-create-a-custom-view)
5. [What is a `Canvas`?](#5-what-is-a-canvas)
6. [What is a `SurfaceView`?](#6-what-is-a-surfaceview)
7. [`RelativeLayout` vs `LinearLayout`](#7-relativelayout-vs-linearlayout)
8. [`ConstraintLayout` optimization (the Cassowary solver)](#8-constraintlayout-optimization-the-cassowary-solver)
9. [The view tree and optimizing layouts / hierarchy depth](#9-the-view-tree-and-optimizing-layouts--hierarchy-depth)
10. [Drawable state lists & 9-patch images](#10-drawable-state-lists--9-patch-images)

### Displaying Lists of Content
11. [`ListView` vs `RecyclerView`](#11-listview-vs-recyclerview)
12. [How does `RecyclerView` work?](#12-how-does-recyclerview-work)
13. [Components of a `RecyclerView`](#13-components-of-a-recyclerview)
14. [`RecyclerView.Adapter` and `RecyclerView.ViewHolder`](#14-recyclerviewadapter-and-recyclerviewviewholder)
15. [What is a `LayoutManager`?](#15-what-is-a-layoutmanager)
16. [Handling multiple view types](#16-handling-multiple-view-types)
17. [`DiffUtil` and `ListAdapter`](#17-diffutil-and-listadapter)
18. [`setHasFixedSize(true)`](#18-sethasfixedsizetrue)
19. [Updating a specific item](#19-updating-a-specific-item)
20. [`RecyclerView` scrolling-performance optimization](#20-recyclerview-scrolling-performance-optimization)
21. [Optimizing nested `RecyclerView`s (shared `RecycledViewPool`)](#21-optimizing-nested-recyclerviews-shared-recycledviewpool)
22. [What is a `SnapHelper`?](#22-what-is-a-snaphelper)

### Dialogs and Toasts
23. [What is a `Dialog`?](#23-what-is-a-dialog)
24. [What is a `Toast`? (and Snackbar)](#24-what-is-a-toast-and-snackbar)
25. [`Dialog` vs `DialogFragment`](#25-dialog-vs-dialogfragment)

### Look and Feel
26. [`Spannable` and `SpannableString`](#26-spannable-and-spannablestring)
27. [Best practices for text in Android](#27-best-practices-for-text-in-android)
28. [Implementing Dark Mode](#28-implementing-dark-mode)

---

## 1. What is a `View` in Android?

A `View` (`android.view.View`) is the **basic building block of the UI**. It occupies a rectangular area on the screen and is responsible for **drawing itself** and **handling events** (touches, key presses, focus). Every widget you see — `TextView`, `Button`, `ImageView`, `EditText`, `CheckBox` — extends `View`.

Each `View` is responsible for three core phases driven by its parent:

- **Measure** (`onMeasure`) — determine the desired size given the parent's constraints (`MeasureSpec`).
- **Layout** (`onLayout`) — position the view (and, for a `ViewGroup`, its children) within the assigned bounds.
- **Draw** (`onDraw`) — render pixels onto a `Canvas`.

A `View` exposes attributes such as `id`, `layout_width`/`layout_height`, `padding`, `background`, `visibility`, and listeners like `setOnClickListener`.

```kotlin
val button = Button(context).apply {
    text = "Tap me"
    setOnClickListener { /* handle click */ }
}
```

**📚 Reference:** AOSP `android.view.View` documentation.

---

## 2. What are `ViewGroup`s and how do they differ from `View`s?

A `ViewGroup` (`android.view.ViewGroup`) is a **special `View` that contains other `View`s** (children), which may themselves be `ViewGroup`s. It is the **base class for layouts and view containers** — `LinearLayout`, `FrameLayout`, `ConstraintLayout`, `RelativeLayout`, `RecyclerView`, `ScrollView`, etc.

| Aspect | `View` | `ViewGroup` |
|---|---|---|
| Role | A single, leaf UI element | An invisible container that arranges children |
| Examples | `TextView`, `Button`, `ImageView` | `LinearLayout`, `ConstraintLayout`, `FrameLayout` |
| Children | None | One or more `View`/`ViewGroup` |
| Key responsibility | Draw itself, handle its own input | Measure, lay out, and dispatch events to children |

A `ViewGroup` adds responsibilities a leaf `View` doesn't have: it measures and positions children (via `LayoutParams`), and it participates in **touch dispatch** through `onInterceptTouchEvent` / `dispatchTouchEvent`.

```xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <TextView ... />   <!-- a View (child) -->
    <Button   ... />   <!-- a View (child) -->
</LinearLayout>          <!-- a ViewGroup (container) -->
```

---

## 3. Difference between `View.GONE` and `View.INVISIBLE`

A `View`'s `visibility` has three states:

| Constant | Visible? | Takes up layout space? | Typical use |
|---|---|---|---|
| `View.VISIBLE` | Yes | Yes | Default |
| `View.INVISIBLE` | No | **Yes** (still measured/laid out) | Hide but keep slot reserved (avoid layout jump) |
| `View.GONE` | No | **No** (treated as if removed) | Remove from layout entirely |

The crucial difference: **`INVISIBLE` keeps the view's space in the layout**, so surrounding views don't shift. **`GONE` removes the view from layout calculations** entirely, so other views collapse into its space.

```kotlin
view.visibility = View.GONE        // hidden, no space
view.visibility = View.INVISIBLE   // hidden, space reserved
view.isVisible = true              // KTX extensions: isVisible / isInvisible / isGone
```

Performance note: toggling between `GONE` and `VISIBLE` triggers a **re-layout** of the parent; toggling `INVISIBLE`/`VISIBLE` does not change layout, only the draw pass — slightly cheaper if you toggle frequently.

**📚 Reference:** https://stackoverflow.com/questions/11556607/android-difference-between-invisible-and-gone

---

## 4. The View lifecycle and how to create a custom `View`

### When do you need a custom view?

When no stock widget gives you the look/behaviour you need, or when you reuse a styled compound widget (e.g. a custom "login field") across many screens. There are two common approaches:

- **Custom-drawn view** — extend `View` and override `onDraw` to paint with a `Canvas` (e.g. a gauge, chart, signature pad).
- **Compound view** — extend a `ViewGroup` (e.g. `ConstraintLayout`) and inflate a layout of existing widgets (e.g. a labelled input row).

### Key lifecycle / override methods

| Method | Purpose |
|---|---|
| Constructors `(Context)`, `(Context, AttributeSet)`, `(Context, AttributeSet, defStyleAttr)` | Created from code / XML; read custom XML attributes here |
| `onAttachedToWindow()` / `onDetachedFromWindow()` | View enters/leaves the window — good for starting/stopping animations, listeners |
| `onMeasure(widthSpec, heightSpec)` | Report your desired size via `setMeasuredDimension()` |
| `onSizeChanged(w, h, oldw, oldh)` | Final size known — cache geometry, build gradients/paths |
| `onLayout(changed, l, t, r, b)` | (`ViewGroup` only) position children |
| `onDraw(Canvas)` | Render pixels. Never allocate objects here |
| `invalidate()` / `requestLayout()` | Trigger a redraw / a re-measure+re-layout |

### Example: a custom circular progress view

```kotlin
class CircleProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    private var progress = 0f  // 0f..1f

    // Allocate Paint ONCE, never inside onDraw
    private val arcPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 24f
        strokeCap = Paint.Cap.ROUND
    }
    private val bounds = RectF()

    init {
        // Read custom attributes declared in res/values/attrs.xml
        context.obtainStyledAttributes(attrs, R.styleable.CircleProgressView).use { ta ->
            arcPaint.color = ta.getColor(R.styleable.CircleProgressView_arcColor, Color.BLUE)
        }
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val size = resolveSize(200, widthMeasureSpec)
        setMeasuredDimension(size, size) // force square
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        val pad = arcPaint.strokeWidth
        bounds.set(pad, pad, w - pad, h - pad)
    }

    override fun onDraw(canvas: Canvas) {
        canvas.drawArc(bounds, -90f, 360f * progress, false, arcPaint)
    }

    fun setProgress(value: Float) {
        progress = value.coerceIn(0f, 1f)
        invalidate() // redraw only
    }
}
```

```xml
<!-- res/values/attrs.xml -->
<resources>
    <declare-styleable name="CircleProgressView">
        <attr name="arcColor" format="color" />
    </declare-styleable>
</resources>
```

**Performance rule:** never allocate (`new`/`apply {}`/`Paint()`) inside `onDraw`; keep `onMeasure`/`onLayout` cheap; call `invalidate()` for redraws and `requestLayout()` only when size changes.

---

## 5. What is a `Canvas`?

A `Canvas` is the **2D drawing surface** handed to you (most often in `View.onDraw(Canvas)`). It holds the draw calls; a **`Paint`** object describes *how* to draw (color, stroke width, anti-aliasing, text size, shaders); and the underlying **`Bitmap`** holds the pixels.

You draw primitives directly:

```kotlin
override fun onDraw(canvas: Canvas) {
    canvas.drawColor(Color.WHITE)                       // background
    canvas.drawCircle(cx, cy, radius, fillPaint)        // shapes
    canvas.drawRect(left, top, right, bottom, strokePaint)
    canvas.drawText("Hello", 50f, 100f, textPaint)
    canvas.drawPath(path, paint)                        // arbitrary paths

    // Save/restore the transformation matrix for rotate/scale/translate
    canvas.withRotation(45f, cx, cy) {                  // KTX helper
        canvas.drawRect(r, paint)
    }
}
```

Common uses: charts, graphs, game/2D rendering, signature pads, custom progress indicators, image editing. You can also draw onto an **off-screen `Bitmap`** by constructing `Canvas(bitmap)`.

By default the `View` system renders with **hardware acceleration** (GPU). A handful of `Canvas` operations are unsupported or have caveats under HW acceleration; in rare cases you set `View.setLayerType(LAYER_TYPE_SOFTWARE, null)`.

**📚 Reference:** AOSP `android.graphics.Canvas`; Amit Shekhar — "What is a Canvas?".

---

## 6. What is a `SurfaceView`?

A `SurfaceView` provides a **dedicated drawing surface embedded inside the view hierarchy** that can be **rendered on a separate (non-UI) thread**. Unlike a normal `View` (drawn on the main thread during the view tree's draw pass), a `SurfaceView` has its own surface that can be updated independently, which is ideal for **frequent, high-throughput rendering**: video playback, camera previews, games, and OpenGL content.

Key points:

- The surface is created/destroyed asynchronously; you observe this via `SurfaceHolder.Callback` (`surfaceCreated` / `surfaceChanged` / `surfaceDestroyed`). Only draw between created and destroyed.
- Drawing is done by locking the canvas: `holder.lockCanvas()` → draw → `holder.unlockCanvasAndPost(canvas)`, typically from a background thread.
- It punches a "hole" in the window; **z-ordering and animation/transformation of a `SurfaceView` are limited** compared to normal views.

```kotlin
class GameView(context: Context) : SurfaceView(context), SurfaceHolder.Callback {
    init { holder.addCallback(this) }

    override fun surfaceCreated(holder: SurfaceHolder) {
        // start a render thread
    }
    override fun surfaceChanged(holder: SurfaceHolder, format: Int, w: Int, h: Int) {}
    override fun surfaceDestroyed(holder: SurfaceHolder) {
        // stop render thread
    }

    private fun renderFrame(holder: SurfaceHolder) {
        val canvas = holder.lockCanvas() ?: return
        try {
            canvas.drawColor(Color.BLACK)
            // draw game state...
        } finally {
            holder.unlockCanvasAndPost(canvas)
        }
    }
}
```

**Related:** `TextureView` behaves like a normal view (can be transformed/animated/alpha-blended) but must be hardware-accelerated and is generally less efficient than `SurfaceView`. For modern camera/graphics, `SurfaceView` is still preferred where you don't need view transforms.

**📚 Reference:** https://developer.android.com/reference/android/view/SurfaceView

---

## 7. `RelativeLayout` vs `LinearLayout`

> Both predate `ConstraintLayout`. For new layouts, prefer **`ConstraintLayout`**, which supersedes both. `RelativeLayout` in particular is now legacy.

**`LinearLayout`** arranges children in a **single row or column** (`orientation="horizontal"|"vertical"`). It supports `layout_weight` to distribute leftover space proportionally.

**`RelativeLayout`** positions children **relative to each other or to the parent** (`layout_toRightOf`, `layout_below`, `alignParentBottom`, etc.), letting you build moderately complex layouts without nesting.

| Aspect | `LinearLayout` | `RelativeLayout` |
|---|---|---|
| Positioning | Sequential (row/column) | Relative to siblings/parent |
| Distribute space | `layout_weight` | No weights |
| Measure passes | **Two passes when using `layout_weight`** (measures children, then re-measures with allotted space) | Generally two passes (horizontal + vertical dependency resolution) |
| Nesting | Often needs nesting for 2D layouts | Reduces nesting vs LinearLayout |
| Status | Supported | Legacy — use ConstraintLayout |

**Performance trade-off:** A `LinearLayout` **without** weights is cheap. **With** `layout_weight`, every weighted child is measured twice, which can be costly when nested. `RelativeLayout` always does roughly two measure passes to resolve dependencies. Deeply nesting either of them multiplies measure/layout cost — which is exactly why `ConstraintLayout` (flat hierarchy, single measure pass in most cases) was introduced.

```xml
<!-- LinearLayout with weights: each weighted child measured twice -->
<LinearLayout android:orientation="horizontal" ...>
    <Button android:layout_width="0dp" android:layout_weight="1" .../>
    <Button android:layout_width="0dp" android:layout_weight="1" .../>
</LinearLayout>
```

---

## 8. `ConstraintLayout` optimization (the Cassowary solver)

`ConstraintLayout` lets you build **large, complex, responsive layouts with a flat (non-nested) view hierarchy** by declaring constraints between view anchors.

### How it works internally — Cassowary

Internally, `ConstraintLayout` translates each constraint into a **system of linear equations and inequalities** over the view anchors (left/top/right/bottom/baseline) and solves them with a **Cassowary-style linear constraint solver**, then applies the computed bounds during `onLayout`. Cassowary is:

- **Incremental** — small changes to constraints don't require re-solving the whole system (useful for animation / `MotionLayout`).
- **Priority/weight-aware** — constraints can have strengths, and `layout_constraintHorizontal_bias` etc. resolve under-constrained cases.

### Why it's faster

- **Flat hierarchy** — replacing nested `LinearLayout`/`RelativeLayout` trees with a single flat container reduces the number of measure/layout/draw recursions.
- Avoids the double measure passes that weighted `LinearLayout`s incur (it can resolve sizes with the solver).

### Optimization features

- **Chains** (`layout_constraintHorizontal_chainStyle`: spread / spread_inside / packed) — distribute a group of views like weights, but flat.
- **Guidelines** — invisible anchors at a fixed dp/percent for aligning multiple views.
- **Barriers** — a virtual edge that moves with the largest of a set of views (great for variable-length text).
- **Groups** — toggle visibility of many views at once.
- **`Layer` / `Flow`** — virtual helpers to arrange/transform groups without extra `ViewGroup`s.
- **`layout_optimizationLevel`** — the solver can skip computing certain constraints (e.g. `direct`, `barrier`) for additional speed.
- **`0dp` (`MATCH_CONSTRAINT`)** with `..._percent`, `..._min`/`_max`, and **dimension ratios** instead of weights.

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    android:layout_width="match_parent" android:layout_height="wrap_content">

    <ImageView android:id="@+id/avatar"
        android:layout_width="48dp" android:layout_height="48dp"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"/>

    <TextView android:id="@+id/title"
        android:layout_width="0dp" android:layout_height="wrap_content"
        app:layout_constraintStart_toEndOf="@id/avatar"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="@id/avatar"/>
</androidx.constraintlayout.widget.ConstraintLayout>
```

**Caveat:** `ConstraintLayout` is not automatically "always faster." For trivial layouts a plain `FrameLayout`/`LinearLayout` may be cheaper; the win comes from **eliminating nesting** in complex layouts.

**📚 Reference:** https://outcomeschool.substack.com/p/constraintlayout-internals-cassowary ; https://android-developers.googleblog.com/2017/08/understanding-performance-benefits-of.html

---

## 9. The view tree and optimizing layouts / hierarchy depth

The **view tree (view hierarchy)** is the tree of `ViewGroup`s and `View`s starting at the window's root (`DecorView`). Rendering walks this tree three times per frame: **measure → layout → draw**. **The deeper and wider the tree, the more work per frame**, and a layout in a frequently re-laid-out spot (e.g. inside a list row) multiplies the cost.

### `ViewTreeObserver`

`ViewTreeObserver` lets you register for global tree events — e.g. `OnGlobalLayoutListener` (fired when layout completes, useful to read a view's measured size), `OnPreDrawListener`, `OnScrollChangedListener`. Remember to remove listeners to avoid leaks.

```kotlin
view.viewTreeObserver.addOnGlobalLayoutListener(
    object : ViewTreeObserver.OnGlobalLayoutListener {
        override fun onGlobalLayout() {
            view.viewTreeObserver.removeOnGlobalLayoutListener(this)
            val w = view.width  // now valid
        }
    }
)
```

### How to optimize layouts / reduce depth

- **Flatten the hierarchy** — replace nested `LinearLayout`/`RelativeLayout` with one `ConstraintLayout`.
- **`<merge>`** — when a custom compound view's root would be redundant with its parent, use `<merge>` to avoid an extra `ViewGroup` level.
- **`<include>`** — reuse layouts without copy/paste (doesn't itself reduce depth, but aids maintainability and pairs with `<merge>`).
- **`ViewStub`** — lazily inflate rarely-used views (error/empty states) so they cost nothing until needed.
- Avoid `layout_weight`-heavy nested `LinearLayout`s (double measure).
- Use **Layout Inspector** and **`adb shell dumpsys gfxinfo`** / GPU rendering profiler to find slow layouts; enable "Show layout bounds" in dev options.
- For lists, use `RecyclerView` (recycling) instead of inflating many views.

**📚 References:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7269208182390951936-dAg3 ; https://developer.android.com/reference/android/view/ViewTreeObserver

---

## 10. Drawable state lists & 9-patch images

### State list drawables — different appearance per button state

Use a **`<selector>`** drawable to present different drawables/colors depending on `View` state (pressed, selected, focused, enabled). Set it as the `android:background` (or `android:src`). **Order matters** — the first matching item wins, so put `state_pressed` before the default.

```xml
<!-- res/drawable/button_bg.xml -->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="true"  android:drawable="@drawable/btn_pressed" />
    <item android:state_selected="true" android:drawable="@drawable/btn_selected" />
    <item android:state_enabled="false" android:drawable="@drawable/btn_disabled" />
    <item android:drawable="@drawable/btn_normal" />  <!-- default -->
</selector>
```

```xml
<Button android:background="@drawable/button_bg" ... />
```

For text colors that change per state, use a **`ColorStateList`** (`res/color/...xml`) the same way.

### 9-patch images

A **9-patch** (`*.9.png`) is a **stretchable bitmap** with 1-px guide borders: the **left/top** black markers define which regions **stretch** (repeat), and the **right/bottom** markers define the **content/padding** region. It **stretches** the marked regions rather than the whole image, so corners and borders stay crisp while the middle expands — perfect for buttons/chat bubbles of variable size. Authored with the `draw9patch` tool / Android Studio's 9-patch editor.

---

## 11. `ListView` vs `RecyclerView`

> `ListView` is **legacy**. For all new lists/grids use `RecyclerView` (or `LazyColumn`/`LazyRow` in Compose).

| Aspect | `ListView` | `RecyclerView` |
|---|---|---|
| View recycling | Optional — only if you implement the ViewHolder pattern manually | **Enforced** via `ViewHolder` |
| Layout | Vertical list only | Pluggable `LayoutManager` (linear, grid, staggered, custom) |
| Item animations | None built-in | `ItemAnimator` (default add/remove/move animations) |
| Item decoration | Limited (`divider`) | `ItemDecoration` (dividers, spacing, custom drawing) |
| Update granularity | `notifyDataSetChanged()` (full redraw) | Fine-grained `notifyItemChanged/Inserted/Removed`, `DiffUtil`, `ListAdapter` |
| Click handling | `setOnItemClickListener` built in | You wire listeners in the ViewHolder |
| Separation of concerns | Monolithic | Adapter / ViewHolder / LayoutManager separated |

**Summary:** `RecyclerView` is more flexible and performant by design (mandatory recycling, pluggable layout, animations, partial updates) but requires more boilerplate. `ListView` is simpler but inefficient and inflexible.

**📚 Reference:** https://stackoverflow.com/questions/26728651/recyclerview-vs-listview

---

## 12. How does `RecyclerView` work?

`RecyclerView` displays a large data set within a small, scrollable window by **recycling a small pool of item views** instead of inflating one per data item.

The flow:

1. The **`LayoutManager`** asks the **`Adapter`** to create just enough `ViewHolder`s to fill the screen (plus a small buffer) via `onCreateViewHolder` (inflates a row layout).
2. For each visible position, `onBindViewHolder(holder, position)` binds data into the **already-inflated** views.
3. As you scroll, a view that leaves the screen is **detached and put into the `Recycler`** (scrap/recycled view pool).
4. When a new position scrolls into view, instead of inflating again, `RecyclerView` **pulls a recycled `ViewHolder` of the same view type from the pool and re-binds it** (`onBindViewHolder`). Inflation — the expensive part — is avoided.

This is why a list of 10,000 rows only ever keeps ~a dozen view instances alive. The pool is keyed by **view type**, so multiple view types each get their own recycled views.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7268187299811606528-u2_w

---

## 13. Components of a `RecyclerView`

| Component | Responsibility |
|---|---|
| **`Adapter`** | Creates `ViewHolder`s, binds data to them, reports item count and view types |
| **`ViewHolder`** | Caches references to a row's child views so `findViewById` runs once |
| **`LayoutManager`** | Measures and positions item views; manages recycling/reuse |
| **`ItemDecoration`** | Draws dividers/spacing/overlays around or between items |
| **`ItemAnimator`** | Animates add/remove/move/change of items (default: `DefaultItemAnimator`) |
| **`RecycledViewPool`** | Holds recycled `ViewHolder`s; can be shared across RecyclerViews |
| **`SnapHelper`** | Snaps scrolling to item boundaries (paging/carousel) |

---

## 14. `RecyclerView.Adapter` and `RecyclerView.ViewHolder`

- **`RecyclerView.ViewHolder`** wraps a single item's view and **holds references to its child views**, so view lookups happen only once (in the holder's constructor) rather than on every bind — a key performance gain.
- **`RecyclerView.Adapter`** is the bridge between data and views. It implements three core callbacks:
  - `onCreateViewHolder(parent, viewType)` — inflate the row layout, return a `ViewHolder`. Called only when a new holder is needed (no recycled one available).
  - `onBindViewHolder(holder, position)` — populate the holder's views with data for `position`. Called often (on every recycle); keep it cheap.
  - `getItemCount()` — total items.

```kotlin
data class User(val id: Long, val name: String)

class UserAdapter(private var items: List<User>) :
    RecyclerView.Adapter<UserAdapter.VH>() {

    class VH(view: View) : RecyclerView.ViewHolder(view) {
        val name: TextView = view.findViewById(R.id.name) // looked up ONCE
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val v = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_user, parent, false)
        return VH(v)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        holder.name.text = items[position].name   // keep light
    }

    override fun getItemCount() = items.size
}
```

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7274733205927182337-hvTG

---

## 15. What is a `LayoutManager`?

A `LayoutManager` is responsible for **measuring and positioning item views** within the `RecyclerView` and for **deciding when to recycle** views that are no longer visible. Swapping the `LayoutManager` changes the list's arrangement without touching the adapter.

Built-in managers:

- **`LinearLayoutManager`** — vertical or horizontal list.
- **`GridLayoutManager`** — grid of N columns/rows; `setSpanSizeLookup` lets items span multiple cells.
- **`StaggeredGridLayoutManager`** — Pinterest-style staggered grid with variable item heights.

You can subclass `RecyclerView.LayoutManager` for fully custom layouts (e.g. circular/carousel).

```kotlin
recyclerView.layoutManager = GridLayoutManager(context, 2)        // 2-column grid
recyclerView.layoutManager = LinearLayoutManager(
    context, RecyclerView.HORIZONTAL, false)                       // horizontal list
```

---

## 16. Handling multiple view types

Override **`getItemViewType(position)`** to return a type constant, then branch on `viewType` in `onCreateViewHolder` to inflate the right layout. The recycler keeps a **separate pool per view type**, so types never get mixed up.

```kotlin
class FeedAdapter(private val items: List<FeedItem>) :
    RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    companion object { const val TYPE_HEADER = 0; const val TYPE_POST = 1 }

    override fun getItemViewType(position: Int) =
        if (items[position].isHeader) TYPE_HEADER else TYPE_POST

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        val inflater = LayoutInflater.from(parent.context)
        return when (viewType) {
            TYPE_HEADER -> HeaderVH(inflater.inflate(R.layout.item_header, parent, false))
            else        -> PostVH(inflater.inflate(R.layout.item_post, parent, false))
        }
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (holder) {
            is HeaderVH -> holder.bind(items[position])
            is PostVH   -> holder.bind(items[position])
        }
    }

    override fun getItemCount() = items.size

    class HeaderVH(v: View) : RecyclerView.ViewHolder(v) { fun bind(i: FeedItem) {} }
    class PostVH(v: View)   : RecyclerView.ViewHolder(v) { fun bind(i: FeedItem) {} }
}
```

For many heterogeneous types, libraries like **`ConcatAdapter`** (combine multiple adapters) or delegate-adapter patterns keep code maintainable.

---

## 17. `DiffUtil` and `ListAdapter`

**`DiffUtil`** computes the **minimal set of changes** between two lists (using an Eugene Myers diff) and dispatches granular `notifyItem*` calls — so only the rows that actually changed are rebound/animated, instead of `notifyDataSetChanged()` redrawing everything.

You implement a `DiffUtil.ItemCallback`:

- `areItemsTheSame` — same identity? (compare IDs)
- `areContentsTheSame` — same content? (compare fields / `equals`)
- (optional) `getChangePayload` — return a payload for partial binds.

The easiest way to use it is **`ListAdapter`**, which runs the diff **on a background thread** and dispatches updates for you:

```kotlin
class UserAdapter : ListAdapter<User, UserAdapter.VH>(DIFF) {

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<User>() {
            override fun areItemsTheSame(a: User, b: User) = a.id == b.id
            override fun areContentsTheSame(a: User, b: User) = a == b
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH { /* ... */ }
    override fun onBindViewHolder(holder: VH, position: Int) {
        val user = getItem(position)  /* ... */
    }

    class VH(v: View) : RecyclerView.ViewHolder(v)
}

// Update simply by submitting a new list; diffing + animations are automatic:
adapter.submitList(newUsers)
```

**Performance benefit:** avoids full rebinds, enables correct add/remove/move animations, and (via `ListAdapter`/`AsyncListDiffer`) keeps diffing off the main thread. For best results submit **new immutable list instances** with stable IDs.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7279435764973686785-pfiQ

---

## 18. `setHasFixedSize(true)`

Call `recyclerView.setHasFixedSize(true)` when **the size of the `RecyclerView` itself does not change** as a result of adapter content changes (i.e. its width/height are `match_parent` or fixed dp, not `wrap_content`).

When true, the `RecyclerView` can **skip requesting a full re-layout of itself** every time items are inserted/removed/changed — it knows its own bounds are stable, so adapter updates only relayout children, not the whole `RecyclerView` and its parents. This is a measurable scroll/update optimization.

**Do not** set it true if the `RecyclerView` uses `wrap_content` (its size genuinely depends on content) — that would produce incorrect sizing.

```kotlin
recyclerView.setHasFixedSize(true)  // RV dimensions are independent of content
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7282252857637007361-thzv/

---

## 19. Updating a specific item

Avoid `notifyDataSetChanged()` (rebinds everything, no animations). Use **granular notifications** so only affected rows update:

```kotlin
// after mutating your backing list:
adapter.notifyItemChanged(position)                 // one item changed
adapter.notifyItemChanged(position, payload)         // partial bind (e.g. just the like count)
adapter.notifyItemInserted(position)
adapter.notifyItemRemoved(position)
adapter.notifyItemMoved(from, to)
adapter.notifyItemRangeChanged(start, count)
```

For **partial updates**, handle the payload in the 3-arg `onBindViewHolder` to rebind only what changed:

```kotlin
override fun onBindViewHolder(holder: VH, position: Int, payloads: MutableList<Any>) {
    if (payloads.isEmpty()) {
        onBindViewHolder(holder, position)          // full bind
    } else {
        // payload present — update just the changed field
        holder.likeCount.text = (payloads.last() as ItemPayload).likes.toString()
    }
}
```

The cleanest modern approach is **`ListAdapter` + `DiffUtil`**: just call `submitList(newList)` and the right granular updates (including payloads) are computed automatically.

---

## 20. `RecyclerView` scrolling-performance optimization

Common techniques to keep scrolling at 60/90/120 fps:

- **Keep `onBindViewHolder` light** — no allocations, no parsing, no formatting in a loop. Pre-compute in the data model.
- **Use the ViewHolder pattern correctly** — look up child views once in the holder; never call `findViewById` in `onBind`.
- **`setHasFixedSize(true)`** when the RV size is content-independent.
- **`DiffUtil` / `ListAdapter`** for granular updates instead of `notifyDataSetChanged()`.
- **Stable IDs** — `setHasStableIds(true)` + `getItemId()` so the recycler can match holders across updates.
- **Flatten item layouts** — use `ConstraintLayout` / fewer nested `ViewGroup`s per row; inflation and measure cost is paid per visible row.
- **Image loading** — use Glide/Coil with proper sizing, caching, and cancel-on-recycle; never decode bitmaps on the main thread.
- **Increase prefetch / cache** — `setItemViewCacheSize(n)`; `LinearLayoutManager` prefetch is on by default and helps with nested lists.
- **Avoid `wrap_content` on the RV** when it forces re-measures.
- **Shared `RecycledViewPool`** for nested lists (see next question).
- **Profile** with Macrobenchmark / FrameTimeline / GPU rendering profiler to find jank.

**📚 Reference:** https://outcomeschool.com/blog/recyclerview-optimization

---

## 21. Optimizing nested `RecyclerView`s (shared `RecycledViewPool`)

A common UI is a **vertical `RecyclerView` whose rows are themselves horizontal `RecyclerView`s** (e.g. a "carousels" home screen). By default **each inner RecyclerView keeps its own recycled-view pool**, so views are inflated redundantly across rows — causing jank.

**Fix:** create **one shared `RecyclerView.RecycledViewPool`** and assign it to every inner RecyclerView, so they all draw from the same set of recycled views.

```kotlin
class OuterAdapter : RecyclerView.Adapter<OuterAdapter.RowVH>() {

    private val sharedPool = RecyclerView.RecycledViewPool()

    inner class RowVH(val innerRv: RecyclerView) : RecyclerView.ViewHolder(innerRv)

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RowVH {
        val innerRv = (LayoutInflater.from(parent.context)
            .inflate(R.layout.row_carousel, parent, false) as RecyclerView)
            .apply {
                layoutManager = LinearLayoutManager(
                    parent.context, RecyclerView.HORIZONTAL, false
                ).apply {
                    // free detached children back to the shared pool immediately
                    recycleChildrenOnDetach = true
                }
                setRecycledViewPool(sharedPool)     // share the pool
            }
        return RowVH(innerRv)
    }

    override fun onBindViewHolder(holder: RowVH, position: Int) {
        holder.innerRv.adapter = InnerAdapter(/* row data */)
    }

    override fun getItemCount() = /* ... */ 0
}
```

Key requirements for it to actually help:

1. **Same view type** across the inner RecyclerViews (the pool is keyed by view type).
2. Set **`recycleChildrenOnDetach = true`** on the inner `LinearLayoutManager` so detached children return to the pool immediately for other rows to reuse.
3. Optionally raise the per-type capacity: `sharedPool.setMaxRecycledViews(viewType, max)` (default is 5).

**📚 Reference:** https://outcomeschool.com/blog/setrecycledviewpool-for-optimizing-nested-recyclerview

---

## 22. What is a `SnapHelper`?

`SnapHelper` attaches to a `RecyclerView` and **snaps scrolling so an item aligns to a target position** when the fling/scroll settles — used for carousels, paging galleries, and "centered card" UIs.

Built-in implementations:

- **`LinearSnapHelper`** — snaps the **nearest item to the center** of the RecyclerView.
- **`PagerSnapHelper`** — snaps **one item per swipe**, like a `ViewPager`.

```kotlin
val snapHelper = PagerSnapHelper()   // or LinearSnapHelper()
snapHelper.attachToRecyclerView(recyclerView)
```

You can subclass `SnapHelper` (or `LinearSnapHelper`) and override `calculateDistanceToFinalSnap`, `findSnapView`, and `findTargetSnapPosition` for custom behavior (e.g. snap to start instead of center).

**📚 Reference:** https://outcomeschool.com/blog/snaphelper

---

## 23. What is a `Dialog`?

A `Dialog` is a **small window that prompts the user** to make a decision or enter information, without filling the screen — typically used for confirmations, alerts, single-choice/multi-choice selections, and short inputs. The base class is `android.app.Dialog`; common subclasses include **`AlertDialog`**, `DatePickerDialog`, `TimePickerDialog`, and (Material) `BottomSheetDialog`.

```kotlin
MaterialAlertDialogBuilder(context)
    .setTitle("Delete file?")
    .setMessage("This cannot be undone.")
    .setPositiveButton("Delete") { dialog, _ -> /* delete */ dialog.dismiss() }
    .setNegativeButton("Cancel") { dialog, _ -> dialog.dismiss() }
    .show()
```

**Caveat:** a raw `Dialog`/`AlertDialog.show()` is tied to its `Activity` and is **not lifecycle-aware** — it does not survive configuration changes (rotation) and can leak the `Activity` if shown from a stale context. For robustness, wrap dialog content in a **`DialogFragment`** (next questions).

**📚 Reference:** https://developer.android.com/guide/topics/ui/dialogs

---

## 24. What is a `Toast`? (and Snackbar)

A `Toast` is a **small, transient, non-modal message** that appears for a short time and **does not take focus** — used for brief, non-critical feedback ("Message sent"). It cannot contain actions and the user can't interact with it.

```kotlin
Toast.makeText(context, "Saved", Toast.LENGTH_SHORT).show()  // or LENGTH_LONG
```

Modern notes (2026):

- On **Android 11+ (API 30+)**, **custom-view toasts (`setView`) are deprecated** for background apps; text toasts are recommended. Toasts from the background are also rate-limited.
- For feedback that needs an **action** (e.g. "Undo") or is tied to a view's lifecycle, prefer **`Snackbar`** (Material), which shows at the bottom, can host an action, and is dismissible.

```kotlin
Snackbar.make(rootView, "Item deleted", Snackbar.LENGTH_LONG)
    .setAction("Undo") { restoreItem() }
    .show()
```

**📚 Reference:** https://developer.android.com/guide/topics/ui/notifiers/toasts

---

## 25. `Dialog` vs `DialogFragment`

| Aspect | `Dialog` | `DialogFragment` |
|---|---|---|
| What it is | A standalone window (`android.app.Dialog`) | A `Fragment` that **hosts** a dialog |
| Lifecycle awareness | None — manual show/dismiss | Tied to `Fragment` lifecycle; integrates with `FragmentManager` |
| Config changes (rotation) | **Dismissed/leaks** unless you manage state yourself | **Automatically recreated/restored** by the system |
| Recommended | Quick, throwaway prompts | **Preferred** for anything non-trivial |
| Back stack / state | Not managed | Managed by `FragmentManager` |

**Why `DialogFragment` is preferred:** it survives configuration changes, properly re-shows after rotation, manages its own lifecycle, and is easier to test and to integrate with ViewModels/navigation. You either override `onCreateDialog` (to return an `AlertDialog`) or `onCreateView` (for a fully custom layout).

```kotlin
class ConfirmDialog : DialogFragment() {
    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog =
        MaterialAlertDialogBuilder(requireContext())
            .setTitle("Confirm")
            .setPositiveButton("OK", null)
            .setNegativeButton("Cancel", null)
            .create()
}

// show it:
ConfirmDialog().show(supportFragmentManager, "confirm")
```

**📚 Reference:** https://stackoverflow.com/questions/7977392/android-dialogfragment-vs-dialog

---

## 26. `Spannable` and `SpannableString`

**Spans** are **ranges of styling applied to portions of text** — color, bold/italic, size, click handlers, images, etc. — letting you style *parts* of a string differently in a single `TextView`.

The text/span class hierarchy:

| Class | Text mutable? | Spans (markup) mutable? | Use when |
|---|---|---|---|
| `String` / `CharSequence` | — | No spans | Plain text |
| `SpannedString` | No | No | Immutable styled text (read-only) |
| **`SpannableString`** | **No** | **Yes** | Text stays the same, styling changes |
| `SpannableStringBuilder` | Yes | Yes | Both text and styling change (e.g. editor) |

> A **`SpannableString` has immutable text but mutable span information** — use it when your text won't change but you want to apply/modify styling.

```kotlin
val text = "Warning: action required"
val spannable = SpannableString(text)

// Make "Warning" red and bold
spannable.setSpan(
    ForegroundColorSpan(Color.RED),
    0, 7, Spannable.SPAN_EXCLUSIVE_EXCLUSIVE
)
spannable.setSpan(
    StyleSpan(Typeface.BOLD),
    0, 7, Spannable.SPAN_EXCLUSIVE_EXCLUSIVE
)

// Make "action" clickable
spannable.setSpan(object : ClickableSpan() {
    override fun onClick(widget: View) { /* handle */ }
}, 9, 15, Spannable.SPAN_EXCLUSIVE_EXCLUSIVE)

textView.text = spannable
textView.movementMethod = LinkMovementMethod.getInstance() // needed for ClickableSpan
```

The `androidx.core` KTX `buildSpannedString { }` / `SpannableStringBuilder` DSL makes this more concise. Common spans: `ForegroundColorSpan`, `BackgroundColorSpan`, `StyleSpan`, `RelativeSizeSpan`, `UnderlineSpan`, `StrikethroughSpan`, `ImageSpan`, `URLSpan`, `ClickableSpan`.

---

## 27. Best practices for text in Android

- **Externalize strings** into `res/values/strings.xml`; never hardcode. Use `getString()`/`@string/...`. This enables **localization (l10n)**.
- **Support translations** with locale-specific `values-<lang>/strings.xml`, and **plurals** via `<plurals>` + `getQuantityString()`.
- **Use `sp` for text size** (scales with the user's font-size accessibility setting) and `dp` for other dimensions.
- **Respect RTL** — use `start`/`end` instead of `left`/`right`, and `android:supportsRtl="true"`.
- **Use string formatting / placeholders** (`%1$s`) rather than concatenation, so word order can be localized.
- **Theme text** with `TextAppearance`/Material type styles instead of per-view font attributes.
- **Use `MaterialTextView`/Material type scale** and `autoSizeTextType` for adaptive sizing.
- **Accessibility:** provide `contentDescription` for non-text, ensure sufficient contrast, and don't disable font scaling.
- For rich/partial styling use **spans** (Q26); for formatted resource text use `<![CDATA[ ]]>` / HTML via `HtmlCompat.fromHtml`.
- Avoid heavy text work on the main thread; for very long text consider **`PrecomputedText`/`AppCompatTextView` with `setTextFuture`** to do measurement off the main thread.

---

## 28. Implementing Dark Mode

Dark mode is implemented via **`DayNight` theming and resource qualifiers** so the system swaps resources automatically based on the night mode.

**1. Use a DayNight theme** (Material 3 themes already extend it):

```xml
<!-- res/values/themes.xml -->
<style name="Theme.MyApp" parent="Theme.Material3.DayNight" />
```

**2. Provide night resources** with the `-night` qualifier — Android picks them automatically when dark mode is on:

```text
res/values/colors.xml          (light)
res/values-night/colors.xml    (dark overrides)
res/drawable/  vs  res/drawable-night/
```

Reference colors via theme attributes (`?attr/colorSurface`, `?android:attr/textColorPrimary`) so they resolve correctly in both modes.

**3. Control the mode at runtime** with `AppCompatDelegate`:

```kotlin
// Follow the system setting (recommended default):
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_FOLLOW_SYSTEM)

// Or let the user force it:
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_YES)  // dark
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO)   // light
```

For **per-app language/theme persistence** on Android 13+ you can also rely on the platform; otherwise persist the user's choice (e.g. DataStore) and apply it at startup.

**Best practices:**
- Don't hardcode colors in layouts — use theme attributes / color resources.
- Test both modes (Android Studio preview has a night toggle; emulator quick settings).
- Handle the night-mode config change (the Activity recreates by default) and avoid flashes.
- Consider **Material 3 dynamic color** (`DynamicColors.applyToActivitiesIfAvailable(app)`) for wallpaper-based theming on Android 12+.

---
