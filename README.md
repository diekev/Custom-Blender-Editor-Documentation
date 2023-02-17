Custom Editor Support in Blender
================================

In Blender an editor is made from a Space, which can contain one or more Regions.
The regions can be the header, footer, UI (colloquially referred to as the side-bar), etc.
The regions are responsible to display data and properties to the user.

This document is for documentation purposes only, it will not create a usable editor
in Blender.

[The code can be found by clicking here, branch "custom_space_type".](https://github.com/diekev/blender)

Creating a Space
----------------

To create an editor, we must first define a Space for it.

```python
class MyEditor(bpy.types.Space):
    # The unique identifier for the editor/space.
    # This will be used for keymaps, gizmos, panels, and any other thing
    # which needs to know which editor/space they are bound to.
    bl_idname = "MY_EDITOR_TYPE"
    # The UI label for the editor.
    bl_label = "My Editor Type"
    # Some optional icon for the UI menus.
    # This will display the Python logo, as this is written in Python.
    # Custom icons are not supported yet.
    bl_icon = 'SCRIPT'

    # Editors can have their own properties.
    # This property will define a query string for the method below to support filtering panel properties
    # (like in the Properties or Outliner spaces).
    # The update function ('self.search_filter_update()') will update the view on each key input.
    query_string: bpy.props.StringProperty(update = lambda self, context: self.search_filter_update())

    # Optional: search_filter_get
    # If you want to add support to find properties within panels in your
    # editor, you can define this function. You are in charge of displaying
    # to the user a text box to enter the query string.
    def search_filter_get(self):
        return self.query_string


    # Optional: eyedropper_color_sample
    # If, for example, your editor displays images in non-linear color spaces,
    # or using some after effects, it can be a good thing to implement a way to
    # sample colors with the eyedropper without the color management or effects.
    # This is what this method is about.
    # If you do not implement this, users can still use the eyedropper in your
    # editor, but the sampled color might not be what they wished for.
    #
    # Params:
    #    region : the region where the eyedropper is
    #    position : the position within the region where the eyedropper is
    # 
    # Return value:
    #    tuple (success, color)
    #        if success is False, the eyedropper will fallback to sampling
    #        the UI draw buffer
    def eyedropper_color_sample(self, region, position):
        return (True, (1.0, 0.0, 1.0, 1.0))


    # Optional: editor_menus_idname
    # This can be used to return the idname of the menu which will be used to
    # populate the search menu (called with the F3 shortcut in the standard
    # Blender keymap).
    # Typically, editors will have one menu which will display all other menus
    # in the editor; said menu's idname would be returned here (see Menus defined
    # at the bottom of this file).
    def editor_menus_idname(self):
        return MY_EDITOR_TYPE_MT_editor_menus.bl_idname


    # Optional: listener
    # This can be used to handled notifications from built-in operators which
    # modified the context in some way. For example, when a CacheFile is created
    # or an object selected.
    # This will be called for each area containing an instance of the Space.
    #
    # Params:
    #   cls: the class type
    #   window: the window where the notification is dispatched
    #   area: the current area (bpy.types.Area) being visited
    #   scene: the current scene
    #   note: the bpy.types.Notifier (see below) which describes the change
    @classmethod
    def listener(cls, window, area, scene, note):
        if note.category == 'SCENE':
            # Redraw the area if the active object or the view layers have changed.
            if note.data in ['OB_ACTIVE', 'LAYER']:
                area.tag_redraw()
```

Adding a Keymap
---------------

You can add keymaps to various regions in your editor the same way you
would add keymaps to existing editors.

```python
# This will just create the keymap, adding mappings to it should be done
# separately, e.g. during add-on registration.
def ensure_region_keymap(window_manager, keymap_name, region_type):
    # Keymaps are put in the addon configs.
    kc = window_manager.keyconfigs.addon

    if not kc:
        return None

    return kc.keymaps.new(name = keymap_name, space_type = "MY_EDITOR_TYPE", region_type = region_type)
```

Creating Regions
----------------

We can add regions to the space. To do so, we simply define subclasses of `bpy.types.Region` for each region we want.

```python
class MyEditorRegion(bpy.types.Region):
    # The unique identifier for this region.
    # This will be used for keymaps, gizmos, panels, and any other thing
    # which needs to know which region they are bound to.
    bl_idname = "MY_EDITOR_REGION_TYPE"
    # The space type where the region should be.
    bl_space_type = 'MY_EDITOR_TYPE'
    # The type of the region.
    # It can be one of the following:
    #    'HEADER'         : the header of the editor, where menus are typically placed
    #    'FOOTER'         : the footer of the editor, where additional data (e.g. statistics)
    #                       can be displayed
    #    'UI'             : the "side bar" panel, where most buttons and properties for
    #                       the editor are typically placed using panels
    #    'WINDOW'         : the main, central region of the editor, where whatever the editor is
    #                       supposed to edit or show is displayed
    #    'NAVIGATION_BAR' : a region to implement some navigation for the main region (e.g. the
    #                       left region of the user preference, which selects what to display)
    #    'TOOLS'          : a region to display tools for the editor
    #    'TOOL_PROPS'     : a region to display tool properties for the editor
    #    'PREVIEW'        : a region to display previews for the editor data
    #    'HUD'            : a region to display properties for the last operator which was
    #                       executed
    #    'EXECUTE'        : a place for buttons to trigger execution of something that
    #                       was set up in other regions
    #    'TOOL_HEADER'    : a region to display tools for the editor
    bl_region_type = 'WINDOW'


    # This will be called whenever the region needs to be initialized
    # or reinitialized (e.g. when resizing the window).
    # Every region needs to be initialized somewhat.
    def init(self, window_manager):
        # First we need to initialize the 2D view of the region.
        # For this, we need to tell what data is expected to be drawn within the region.
        #  STANDARD   : view with default parameters. This can be used if you do not intend
        #               to draw panels in the view, but rather some other things (e.g. an 
        #               image, or some custom widgets)
        #  LIST       : view to display lists (e.g. the outliner)
        #  STACK      : view for lists where items are added at the top
        #  HEADER     : view for header or footer regions
        #  PANELS_UI  : view for regions containing panels
        #
        # For regions which will only display panels, a call to
        # `Region.panels_init(window_manager, self)` should be used instead.
        bpy.types.Region.view2d_init(self, 'STANDARD')

        # Then you can setup or initialize additional data, some of your own,
        # or some for Blender.

        # Keymap support for this region can be added thusly:
        keymap = ensure_region_keymap(window_manager, 'MyRegionKeymap', 'WINDOW')
        if keymap:
            self.add_keymap(keymap)

        # Gizmo support for this region can be ensured by "refreshing" the region's
        # gizmo map:
        self.gizmos_tag_refresh()

        # Regions are generally only redrawn when the mouse is hovering them.
        # However, if you want this region to be redrawn when some specific event
        # happens (in your editor or in another editor) you should here setup some
        # message bus subscribers.
        # For example, if you want to redraw the region when a node graph has a new
        # active node:
        bpy.msgbus.subscribe_rna(
            key=(bpy.types.Nodes, "active"),
            owner=self,
            args=(self, ), # 'args' will be passed to 'notifiy'
            notify=lambda region: region.tag_redraw(),
        )


    # This is called whenever the region has to be drawn.
    # For headers and footers, a subclass of `bpy.types.Header` needs to be used instead.
    # If the region only has panels, this can be left out.
    # Make sure this is as fast as possible, as the region might be redrawn
    # on every mouse move.
    def draw(self, context):
        # First you can clear the background of the region, which will
        # use the theme's background color here (custom editors inherit their
        # theme from the properties editor at the moment). If you skip this, the
        # background will be black.
        self.theme_clear_color('BACK')

        # Here you can let handlers (see Space.draw_handlers_add) draw before you do.
        # This uses 'PRE_VIEW', but can also be 'BACKDROP', depending on what semantics
        # you prefer.
        self.draw_handlers(context, 'PRE_VIEW')

        # Then you can draw whatever you want using the GPU module.

        # Then you can let post draw handlers do their drawing.
        self.draw_handlers(context, 'POST_VIEW')

        # Finally, if you support gizmos, you should draw them.
        self.gizmos_draw(context)

    
    # Optional: listener
    # This can be used to handled notifications from built-in operators which
    # modified the context in some way. For example, when a CacheFile is created
    # or an object selected.
    # This will be called for each instance of the Region.
    #
    # Params:
    #   cls: the class type
    #   window: the window where the notification is dispatched
    #   area: the current area (bpy.types.Area) being visited, this can be null if
    #         the region is not part of an area 
    #   region: the current region being visited
    #   scene: the current scene
    #   note: the bpy.types.Notifier (see below) which describes the change
    @classmethod
    def listener(cls, window, area, region, scene, note):
        if note.category == 'SCENE':
            # Redraw the region if the active object or the view layers have changed.
            if note.data in ['OB_ACTIVE', 'LAYER']:
                region.tag_redraw()
```

UI for Headers and Footers
--------------------------

To draw the header and footer regions, a subclass of `bpy.types.Header`
needs to used.

```python
class MyHeaderRegionUI(bpy.types.Header):
    # Use 'FOOTER' for a footer
    bl_region_type = 'HEADER'

    def draw(self, context):
        layout = self.layout

        # Access the editor.
        editor = context.space_data

        # Displays the editor selection drop-down menu.
        layout.template_header()

        # Displays the "View" menu defined below.
        layout.menu("MY_EDITOR_TYPE_MT_view_menu")

        # Displays an input for the query string.
        layout.separator_spacer()
        layout.prop(editor, "query_string", icon='VIEWZOOM', text="")
        layout.separator_spacer()
```

Default Properties
------------------

Properties to toggle visibility of regions will be automatically added to the
editor, which can be put in a menu.

Here are the names of those properties for each relevant region type.

| Region Type | Property Name |
|:----------|:--------------------|
| FOOTER | show_region_footer |
| HEADER | show_region_header |
| HUD | show_region_hud |
| UI | show_region_ui |
| TOOL_HEADER | show_region_tool_header |
| TOOL_PROPS | show_region_tool_props |
| TOOLS | show_region_toolbar |

You can add keymaps for each of these, e.g.:

```python
    item = keymap.keymap_items.new("wm.context_toggle", type = 'N', value = 'PRESS')
    item.properties.data_path = 'space_data.show_region_ui'
```

```python
# Menu to expose the 'show_region_ui' property.
class MY_EDITOR_TYPE_MT_view_menu(bpy.types.Menu):
    bl_label = "View"

    def draw(self, context):
        layout = self.layout
        editor = context.space_data

        layout.prop(editor, "show_region_ui")
```

Editor Menus
------------

Meta-menu for `Space.editor_menus_idname()`

```python
class MY_EDITOR_TYPE_MT_editor_menus(bpy.types.Menu):
    def draw(self, context):
        layout = self.layout

        # Menus here will be accessible from the search menu operator (F3).
        layout.menu("MY_EDITOR_TYPE_MT_view_menu")
```

Notifiers (bpy.types.Notifier)
------------------------------

Notifiers hold data describing where a change happened and what has changed.

### Properties

#### category (Notifier.category)

Enumeration, describe which parent type (e.g. ID, Space, Window Manager) where the change occured.

Possible values :

| Name | Relates to |
|:----------|:--------------------|
| WM | the Window Manager |
| WINDOW | the Window |
| WORKSPACE | the Workspace |
| SCREEN | a Screen ID |
| SCENE | the Scene |
| OBJECT | an Object ID |
| MATERIAL | a Material ID |
| TEXTURE | a Texture ID |
| LAMP | a light object |
| COLLECTION | a Collection ID |
| IMAGE | an Image ID |
| BRUSH | a Brush ID |
| TEXT | a Text ID |
| WORLD | a World ID |
| ANIMATION | animation data |
| SPACE | a Space |
| GEOM | geometry data |
| NODE | a Node or the node editor |
| ID | some ID |
| PAINT_CURVE | a paint curve |
| MOVIE_CLIP | a Movie Clip ID |
| MASK | a Mask ID |
| GREASE_PENCIL | a Grease Pencil object |
| LINESTYLE | a Line Style object |
| CAMERA | a Camera object |
| LIGHTPROBE | a light probe |
| ASSET | an Asset ID |
| VIEWER_PATH | a viewer path |

In some cases, the category can be confusing. For example, if an object is added, then the category is 'SCENE' and not 'OBJECT' as the Scene has a new object.

#### subtype (Notifier.subtype)

Enumeration, gives a precision to the category.

| Name | Description |
|:----------|:--------------------|
| MODE_OBJECT | Changed happened in object mode |
| EDITMODE_MESH | Changed happened in the edit mode of a Mesh object |
| EDITMODE_CURVE | Changed happened in the edit mode of a legacy Curve object |
| EDITMODE_CURVES | Changed happened in the edit mode of a Curves object |
| EDITMODE_SURFACE | Changed happened in the edit mode of a NURBS object |
| EDITMODE_TEXT | Changed happened in the edit mode of a Text object |
| EDITMODE_MBALL | Changed happened in the edit mode of a Meta Ball object |
| EDITMODE_LATTICE | Changed happened in the edit mode of a Lattice object |
| EDITMODE_ARMATURE | Changed happened in the edit mode of an Armature object |
| MODE_POSE | Changed happened in Pose mode |
| MODE_PARTICLE | Changed happened in Particles edit mode |
| VIEW3D_GPU | Changed happened when the 3D view was being drawn regularly |
| VIEW3D_SHADING | Changed happened when the 3D view was being rendered in preview |
| LAYER_COLLECTION | Changed happened when editing a view layer |

#### action (Notifier.action)

Enumeration, the action being performed.

| Name | Description |
|:----------|:--------------------|
| EDITED | Something was edited |
| EVALUATED | Something was evaluated |
| ADDED | Something was added |
| REMOVED | Something was removed |
| RENAME | Something was renamed |
| SELECTED | Something was evaluated |
| ACTIVATED | Something was set as active |
| PAINTING | Something was painted on |
| JOB_FINISHED | A job finished (e.g. a bake, the loading of an Alembic file is done) |

#### data (Notifier.data)

Enumeration, the data which was modified.

The possible values for the data depend on which category the changed happened in. Some categories have no data, and therefore for those the only value is 'NONE'.

If category is 'WM'.

| Name | Description |
|:----------|:--------------------|
| FILEREAD | |
| FILESAVE | |
| DATACHANGED | |
| HISTORY | |
| JOB | |
| UNDO | |
| XR_DATA_CHANGED | |
| LIB_OVERRIDE_CHANGED | |

If category is 'SCREEN'.

| Name | Description |
|:----------|:--------------------|
| LAYOUTBROWSE | |
| LAYOUTDELETE | |
| ANIMPLAY | |
| GPENCIL | |
| LAYOUTSET | |
| SKETCH | |
| WORKSPACE_SET | |
| WORKSPACE_DELETE | |

If category is 'SCENE'.

| Name | Description |
|:----------|:--------------------|
| SCENEBROWSE | |
| MARKERS | |
| FRAME | |
| RENDER_OPTIONS | |
| NODES | |
| SEQUENCER | |
| OB_ACTIVE | |
| OB_SELECT | |
| OB_VISIBLE | |
| OB_RENDER | |
| MODE | |
| RENDER_RESULT | |
| COMPO_RESULT | |
| KEYINGSET | |
| TOOLSETTINGS | |
| LAYER | |
| FRAME_RANGE | |
| TRANSFORM_DONE | |
| WORLD | |
| LAYER_CONTENT | |

If category is 'OBJECT'.

| Name | Description |
|:----------|:--------------------|
| TRANSFORM | |
| OB_SHADING | |
| POSE | |
| BONE_ACTIVE | |
| BONE_SELECT | |
| DRAW | |
| MODIFIER | |
| KEYS | |
| CONSTRAINT | |
| PARTICLE | |
| POINTCACHE | |
| PARENT | |
| LOD | |
| DRAW_RENDER_VIEWPORT | |
| SHADERFX | |
| DRAW_ANIMVIZ | |

If category is 'MATERIAL'.

| Name | Description |
|:----------|:--------------------|
| SHADING | |
| SHADING_DRAW | |
| SHADING_LINKS | |
| SHADING_PREVIEW | |

If category is 'LAMP'.

| Name | Description |
|:----------|:--------------------|
| LIGHTING | |
| LIGHTING_DRAW | |

If category is 'WORLD'.

| Name | Description |
|:----------|:--------------------|
| WORLD_DRAW | |

If category is 'TEXT'.

| Name | Description |
|:----------|:--------------------|
| CURSOR | |
| DISPLAY | |

If category is 'ANIMATION'.

| Name | Description |
|:----------|:--------------------|
| KEYFRAME | |
| KEYFRAME_PROP | |
| ANIMCHAN | |
| NLA | |
| NLA_ACTCHANGE | |
| FCURVES_ORDER | |
| NLA_ORDER | |

If category is 'GPENCIL'.

| Name | Description |
|:----------|:--------------------|
| GPENCIL_EDITMODE | |

If category is 'GEOM'.

| Name | Description |
|:----------|:--------------------|
| SELECT | |
| DATA | |
| VERTEX_GROUP | |

If category is 'SPACE'.

| Name | Description |
|:----------|:--------------------|
| SPACE_CONSOLE | |
| SPACE_INFO_REPORT | |
| SPACE_INFO | |
| SPACE_IMAGE | |
| SPACE_FILE_PARAMS | |
| SPACE_FILE_LIST | |
| SPACE_ASSET_PARAMS | |
| SPACE_NODE | |
| SPACE_OUTLINER | |
| SPACE_VIEW3D | |
| SPACE_PROPERTIES | |
| SPACE_TEXT | |
| SPACE_TIME | |
| SPACE_GRAPH | |
| SPACE_DOPESHEET | |
| SPACE_NLA | |
| SPACE_SEQUENCER | |
| SPACE_NODE_VIEW | |
| SPACE_CHANGED | |
| SPACE_CLIP | |
| SPACE_FILE_PREVIEW | |
| SPACE_SPREADSHEET | |

If category is 'ASSET'.

| Name | Description |
|:----------|:--------------------|
| ASSET_LIST | |
| ASSET_LIST_PREVIEW | |
| ASSET_LIST_READING | |
| ASSET_CATALOGS | |