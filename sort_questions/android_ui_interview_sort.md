# Android UI / View System — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_ui_interview.md`.

---

## 1. What is a `View` in Android?

**View** means a rectangular widget class (`android.view.View`) that occupies a screen area, draws itself, and handles user input events.

The basic UI building block (`android.view.View`) — a rectangular area responsible for drawing itself and handling events. Every widget (`TextView`, `Button`, etc.) extends it. It runs three parent-driven phases: measure (`onMeasure`), layout (`onLayout`), and draw (`onDraw`).

---



## 2. What are `ViewGroup`s and how do they differ from `View`s?

**ViewGroup** means a container class (`android.view.ViewGroup`) that hosts, measures, and positions a collection of child View objects.

A `ViewGroup` is a special `View` that contains children — the base class for layouts/containers (`LinearLayout`, `ConstraintLayout`, `RecyclerView`). A leaf `View` just draws itself and handles its own input; a `ViewGroup` additionally measures, positions (via `LayoutParams`), and dispatches touch events to children.

---



## 3. Difference between `View.GONE` and `View.INVISIBLE`

**View.GONE vs View.INVISIBLE** means the difference between hiding a view and removing its space from the layout (GONE), and hiding a view while keeping its space allocated (INVISIBLE).

`INVISIBLE` hides the view but **keeps its layout space** (surrounding views don't shift). `GONE` hides it and **removes it from layout** entirely (others collapse into its space). Toggling GONE/VISIBLE triggers a re-layout; INVISIBLE/VISIBLE only a redraw.

---



## 4. The View lifecycle and how to create a custom `View`

**View lifecycle** means the sequence of callbacks (like `onMeasure`, `onLayout`, and `onDraw`) that govern how a view is measured, positioned, and rendered on the screen.

Two approaches: custom-drawn (extend `View`, override `onDraw`) or compound (extend a `ViewGroup`, inflate child widgets). Key methods: constructors (read XML attrs), `onAttached/DetachedFromWindow`, `onMeasure` (call `setMeasuredDimension`), `onSizeChanged`, `onLayout` (ViewGroup), `onDraw`, plus `invalidate()` (redraw) / `requestLayout()` (re-measure). **Never allocate objects in `onDraw`.**

---



## 5. What is a `Canvas`?

**Canvas** means a drawing surface class (`android.graphics.Canvas`) that holds the actual 2D drawing calls to render shapes, text, or bitmaps.

The 2D drawing surface handed to `onDraw(Canvas)`. A `Paint` describes how to draw (color, stroke, anti-aliasing); the underlying `Bitmap` holds pixels. You call `drawCircle`/`drawRect`/`drawText`/`drawPath`, and save/restore transforms. Used for charts, games, signature pads. Rendered with hardware acceleration by default.

---



## 6. What is a `SurfaceView`?

**SurfaceView** means a specialized view that embeds a dedicated drawing surface inside the view hierarchy to perform background-thread rendering.

A dedicated drawing surface embedded in the view hierarchy that can be rendered on a separate (non-UI) thread — ideal for high-throughput rendering (video, camera preview, games). Observe its surface via `SurfaceHolder.Callback`; draw with `lockCanvas()` → draw → `unlockCanvasAndPost()`. Z-ordering and transforms are limited. `TextureView` can be transformed/animated but is less efficient.

---



## 7. `RelativeLayout` vs `LinearLayout`

**RelativeLayout vs LinearLayout** means the choice between arranging child views relative to each other or parent boundaries (RelativeLayout), and arranging child views in a single vertical or horizontal row (LinearLayout).

`LinearLayout` arranges children in a single row/column and supports `layout_weight` (but weighted children are measured twice). `RelativeLayout` positions children relative to siblings/parent, reducing nesting, but does ~two measure passes. Both are superseded by `ConstraintLayout` (flat hierarchy); RelativeLayout is legacy.

---



## 8. `ConstraintLayout` optimisation (the Cassowary solver)

**ConstraintLayout** means a flexible, flat layout container that uses constraint equations and the Cassowary solver to align views without nested view groups.

Builds large, responsive layouts with a flat (non-nested) hierarchy by solving constraints as a system of linear equations via a Cassowary-style solver (incremental, priority-aware). It's faster because it flattens the hierarchy and avoids weighted double-measures. Features: chains, guidelines, barriers, groups, `0dp` (MATCH_CONSTRAINT), bias, dimension ratios. Not always faster for trivial layouts.

---



## 9. The view tree and optimizing layouts / hierarchy depth

**View tree optimisation** means reducing layout nesting to avoid deep recursion during measure and layout phases which can cause dropped frames.

The view hierarchy is walked measure→layout→draw each frame; deeper/wider trees cost more. `ViewTreeObserver` gives global tree events (e.g. `OnGlobalLayoutListener` to read measured size — remove listeners to avoid leaks). Optimise by flattening (ConstraintLayout), `<merge>`, `<include>`, `ViewStub` (lazy inflation), avoiding weighted nested LinearLayouts, and using `RecyclerView` for lists.

---



## 10. Drawable state lists & 9-patch images

**StateListDrawable** means an XML-defined drawable resource that changes its background graphic dynamically based on the target view's state (like pressed or selected).

A `<selector>` drawable shows different drawables per view state (pressed, selected, disabled) — order matters (first match wins; put the default last). Use `ColorStateList` for per-state text colors. A **9-patch** (`*.9.png`) is a stretchable bitmap with 1-px guides: left/top markers define stretchable regions, right/bottom define content/padding — keeps corners crisp.

---



## 11. `ListView` vs `RecyclerView`

**ListView vs RecyclerView** means the choice between a simple, legacy list widget that recreates views frequently (ListView), and a highly extensible, modular list container that recycles views (RecyclerView).

`RecyclerView` enforces view recycling (ViewHolder), has a pluggable `LayoutManager`, built-in item animations and decorations, fine-grained updates (`DiffUtil`/`ListAdapter`), and separated concerns — more flexible and performant but more boilerplate. `ListView` is simpler but inefficient and inflexible (legacy).

---



## 12. How does `RecyclerView` work?

**RecyclerView layout flow** means displaying data by reusing a small pool of views cached in scrap, recycle, and cache pools to avoid allocations during scrolling.

It displays large data in a small scrollable window by recycling a small pool of item views. The `LayoutManager` asks the adapter to create just enough `ViewHolder`s (`onCreateViewHolder` inflates) and binds via `onBindViewHolder`. Views scrolling off go into the recycler pool (keyed by view type) and are re-bound for new positions, avoiding re-inflation.

---



## 13. Components of a `RecyclerView`

**RecyclerView architecture** means the collaboration of Adapter, ViewHolder, LayoutManager, Recycler, ItemAnimator, and ItemDecoration to render lists.

Adapter (creates/binds ViewHolders, reports count/types), ViewHolder (caches child view references), LayoutManager (measures/positions + recycling), ItemDecoration (dividers/spacing), ItemAnimator (add/remove/move animations), RecycledViewPool (holds recycled holders, shareable), SnapHelper (snaps scroll to items).

---



## 14. `RecyclerView.Adapter` and `RecyclerView.ViewHolder`

**Adapter vs ViewHolder** means the distinction between a class that binds data items to views (Adapter), and a class that caches view references to avoid `findViewById` calls (ViewHolder).

A `ViewHolder` wraps an item's view and caches child references (lookup once in its constructor). The `Adapter` bridges data and views via `onCreateViewHolder` (inflate, called only when no recycled holder), `onBindViewHolder` (bind data, called often — keep cheap), and `getItemCount()`.

---



## 15. What is a `LayoutManager`?

**LayoutManager** means a RecyclerView component responsible for measuring, positioning, and determining when to recycle item views.

Responsible for measuring/positioning item views and deciding when to recycle them; swapping it changes the arrangement without touching the adapter. Built-in: `LinearLayoutManager` (list), `GridLayoutManager` (grid, with span lookup), `StaggeredGridLayoutManager` (variable-height grid). Subclass for custom layouts.

---



## 16. Handling multiple view types

**Multiple view types** means configuring a RecyclerView adapter to inflate different XML layouts for different data items based on position-based type constants.

Override `getItemViewType(position)` to return a type constant, then branch on `viewType` in `onCreateViewHolder` to inflate the right layout (the pool keeps a separate set per type). For many types, use `ConcatAdapter` or delegate-adapter patterns.

---



## 17. `DiffUtil` and `ListAdapter`

**DiffUtil** means an algorithm utility class that calculates the minimum difference between two lists and dispatches granular updates to the RecyclerView.

`DiffUtil` computes the minimal change set between two lists (Myers diff) and dispatches granular `notifyItem*` calls instead of `notifyDataSetChanged()`. Implement `areItemsTheSame` (IDs), `areContentsTheSame` (fields), optional `getChangePayload`. `ListAdapter` runs the diff on a background thread — just call `submitList(newList)`. Submit new immutable lists with stable IDs.

---



## 18. `setHasFixedSize(true)`

**setHasFixedSize(true)** means optimizing RecyclerView by telling it that its width and height do not depend on adapter item contents, preventing full layout invalidation.

Set it when the RecyclerView's own size doesn't change with content (fixed/`match_parent`, not `wrap_content`). It then skips a full self re-layout on adapter changes — adapter updates only relayout children. Don't use with `wrap_content`.

---



## 19. Updating a specific item

**Granular updates** means calling specific adapter notify methods (like `notifyItemChanged`) to animate and update only affected items instead of calling `notifyDataSetChanged()`.

Avoid `notifyDataSetChanged()`. Use granular notifications: `notifyItemChanged(pos)`, `notifyItemInserted/Removed/Moved`, `notifyItemRangeChanged`, and `notifyItemChanged(pos, payload)` for partial binds (handle payloads in the 3-arg `onBindViewHolder`). Cleanest modern approach: `ListAdapter` + `DiffUtil` with `submitList`.

---



## 20. `RecyclerView` scrolling-performance optimisation

**RecyclerView scroll optimisation** means techniques like using image libraries, avoiding complex layouts, prefetching items, and keeping bind methods fast to avoid dropped frames.

Keep `onBindViewHolder` light (no allocations/parsing), use the ViewHolder pattern, `setHasFixedSize(true)`, `DiffUtil`/`ListAdapter`, stable IDs, flatten item layouts (ConstraintLayout), load images with Glide/Coil (sized, cached, cancel-on-recycle), increase prefetch/`setItemViewCacheSize`, share a `RecycledViewPool` for nested lists, and profile for jank.

---



## 21. Optimizing nested `RecyclerView`s (shared `RecycledViewPool`)

**Nested RecyclerView optimisation** means sharing a single `RecycledViewPool` between outer and inner RecyclerViews to prevent redundant view holder creation.

By default each inner RecyclerView keeps its own pool, causing redundant inflation (jank). Create one shared `RecycledViewPool` and assign it to every inner RecyclerView via `setRecycledViewPool(pool)`. Requires the same view type across them, `recycleChildrenOnDetach = true` on the inner `LinearLayoutManager`, and optionally a higher per-type capacity.

---



## 22. What is a `SnapHelper`?

**SnapHelper** means a helper utility that snaps RecyclerView item scroll offsets to align elements to specific coordinates.

Attaches to a RecyclerView and snaps scrolling so an item aligns to a target when the fling settles (carousels, paging). `LinearSnapHelper` snaps the nearest item to center; `PagerSnapHelper` snaps one item per swipe (like ViewPager). Subclass for custom snap behaviour.

---



## 23. What is a `Dialog`?

**Dialog** means a floating window component that prompts users for input or choices without filling the full screen.

A small window that prompts the user without filling the screen (confirmations, alerts, inputs). Base class `android.app.Dialog`; subclasses include `AlertDialog`, `DatePickerDialog`, `BottomSheetDialog`. A raw `Dialog` is not lifecycle-aware (dismissed/leaks on rotation) — wrap content in a `DialogFragment` for robustness.

---



## 24. What is a `Toast`? (and Snackbar)

**Toast vs Snackbar** means the choice between a system-controlled, transient modal message (Toast), and an app-contained message that can host action callbacks and support swipes (Snackbar).

A small, transient, non-modal message that doesn't take focus, for brief non-critical feedback. On Android 11+ custom-view toasts are deprecated for background apps and toasts are rate-limited. For an action ("Undo") or view-tied feedback, prefer **Snackbar** (Material), which is dismissible and can host an action.

---



## 25. `Dialog` vs `DialogFragment`

**Dialog vs DialogFragment** means the choice between a raw floating window that is dismissed or leaked on configuration change (Dialog), and a lifecycle-aware fragment wrapper (DialogFragment).

A `Dialog` is a standalone window with no lifecycle awareness (dismissed/leaks on rotation). A `DialogFragment` hosts a dialog, is tied to the Fragment lifecycle and `FragmentManager`, and is automatically recreated/restored across config changes. Prefer `DialogFragment` — override `onCreateDialog` (AlertDialog) or `onCreateView` (custom layout).

---



## 26. `Spannable` and `SpannableString`

**Spannable** means a text container interface that allows applying different styling ranges (like bold, size, or click actions) to specific character ranges.

Spans are ranges of styling (color, bold, size, click, image) applied to parts of text in one `TextView`. Hierarchy: `SpannedString` (immutable text + spans), **`SpannableString`** (immutable text, mutable spans), `SpannableStringBuilder` (both mutable). Apply via `setSpan(span, start, end, flag)`; `ClickableSpan` needs `LinkMovementMethod`.

---



## 27. Best practices for text in Android

**Text resource best practices** means externalising UI text in `strings.xml` to support localization and using pre-computed text layouts to avoid UI thread lag.

Externalize strings (`strings.xml`) for localization; support translations + `<plurals>`; use `sp` for text size and `dp` otherwise; respect RTL (`start`/`end`, `supportsRtl`); use placeholders not concatenation; theme via `TextAppearance`/Material styles; provide `contentDescription` and good contrast; use spans for rich styling and `PrecomputedText` for long text off the main thread.

---



## 28. Implementing Dark Mode

**Dark mode implementation** means using `DayNight` themes and alternative resource qualifiers (`values-night`) to automatically swap resources based on system preferences.

Use a `DayNight` theme (Material 3 already extends it) and `-night` resource qualifiers so Android swaps colors/drawables automatically; reference colors via theme attributes (`?attr/colorSurface`). Control the mode with `AppCompatDelegate.setDefaultNightMode(...)`. Best practices: don't hardcode colors, test both modes, handle the night-mode config change, and consider Material 3 dynamic color.

