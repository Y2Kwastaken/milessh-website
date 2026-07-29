---
title: 'MenuKit'
author: 'miles_dev'
---

## Architectural Overview

MenuKit is a state-driven inventory GUI abstraction built specifically for the PaperMC ecosystem. It replaces monolithic `InventoryClickEvent` listeners with a localized, functional callback system and introduces state-retaining virtual pages natively tied to Paper's Component and Item APIs.

The framework is constructed around these core pillars:

* **SlotMenu**: The base class linking a Bukkit `InventoryView` to a `PagedInventory` state manager. View creation is handled via `MenuType` abstractions rather than raw Bukkit inventory calls.
* **PagedInventory**: An internal wrapper mapping multiple virtual pages to a single, static frontend size.
* **MenuRecipe**: A declarative layout engine mapping characters to GUI slots.
* **MenuSlot / MenuStack**: Functional wrappers carrying `ItemStack` data and interaction callbacks.

---

## Installation

MenuKit is distributed via the Miles Repository. It is split into core functionality and the optional string-layout engine.

**Gradle (Kotlin DSL)**
```kotlin
repositories {
    maven("https://maven.miles.sh/snapshots")
}

dependencies {
    implementation("sh.miles.menukit:menukit-core:2.1.0-SNAPSHOT")
    implementation("sh.miles.menukit:menukit-strings:2.0.0-SNAPSHOT") // Optional: For MenuRecipe layouts
}
```

**Maven**
```xml
<repository>
    <id>miles-repos-snapshots</id>
    <url>https://maven.miles.sh/snapshots</url>
</repository>

<dependency>
    <groupId>sh.miles.menukit</groupId>
    <artifactId>menukit-core</artifactId>
    <version>2.1.0-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>sh.miles.menukit</groupId>
    <artifactId>menukit-strings</artifactId>
    <version>2.0.0-SNAPSHOT</version>
</dependency>
```

## Framework Initialization

MenuKit must be initialized within your plugin's lifecycle to register its internal `SlotMenuManager` and lifecycle listeners.

```java
import org.bukkit.plugin.java.JavaPlugin;
import sh.miles.menukit.MenuKit;

public class MyPlugin extends JavaPlugin {
    
    @Override
    public void onEnable() {
        MenuKit.INSTANCE.start(this);
    }

    @Override
    public void onDisable() {
        MenuKit.INSTANCE.stop();
    }
}
```

---

## Creating Menus

MenuKit supports two distinct paradigms for menu creation: Object-Oriented (extending `SlotMenu`) and Functional (`SlotMenuFactory`). Both rely on `MenuType` for standardized view generation.

### The Object-Oriented Approach
Best suited for complex menus with dedicated state, extensive pagination, or reusable layouts.

```java
import sh.miles.menukit.menu.SlotMenu;
import sh.miles.menukit.menu.MenuType;
import org.bukkit.entity.Player;
import org.bukkit.inventory.InventoryView;
import net.kyori.adventure.text.Component;

public class ProfileMenu extends SlotMenu<InventoryView> {

    public ProfileMenu(Player player) {
        // Initialize with a 27 slot view and 1 virtual page
        super(player, p -> MenuType.GENERIC_9X3.create(p, Component.text("Player Profile")), 1);
    }

    @Override
    protected void reload(InventoryView view) {
        // Initialization and layout logic goes here
        // Called automatically upon menu.open()
    }
}
```

### The Functional Approach
Best suited for simple, stateless interactions like admin tools or quick confirmations.

```java
import sh.miles.menukit.menu.SlotMenuFactory;
import sh.miles.menukit.menu.MenuType;
import org.bukkit.entity.Player;
import org.bukkit.inventory.InventoryView;
import net.kyori.adventure.text.Component;

public class AdminTools {

    public static void openQuickMenu(Player player) {
        SlotMenuFactory<InventoryView> factory = new SlotMenuFactory<>(
            p -> MenuType.GENERIC_9X1.create(p, Component.text("Admin Actions")), 1
        );

        factory.create(player, (view, inventory) -> {
            // Initialization logic goes here
        }).open();
    }
}
```

---

## Defining Items & Callbacks

MenuKit utilizes `MenuSlot` and `MenuStack` to bind `ItemStack` instances to `MenuEventCallback` executions. It natively supports Paper's `ItemType` and `DataComponentTypes`.

### Using MenuSlot Builders
When defining items manually within a `reload` block, use the `createSlot` helper to bind interaction logic directly to an index.

```java
import org.bukkit.inventory.ItemType;
import io.papermc.paper.datacomponent.DataComponentTypes;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import sh.miles.menukit.menu.MenuEventCallback;

@Override
protected void reload(InventoryView view) {
    this.getInventory().setItem(createSlot(builder -> builder
            .index(4)
            .content(ItemType.DIAMOND, item -> {
                item.setData(DataComponentTypes.ITEM_NAME, Component.text("Claim Reward", NamedTextColor.AQUA));
            })
            .click(e -> {
                e.cancel();
                e.getPlayer().getInventory().addItem(ItemType.DIAMOND.createItemStack());
                e.getPlayer().closeInventory();
            })
            // Using pre-defined constants to disable specific interactions
            .drag(MenuEventCallback.DRAG_CANCEL)
    ));
}
```

### Using MenuStack
`MenuStack` is an immutable record utilized primarily for mapping repetitive items or integrating with the `MenuRecipe` system. The library provides utility factory methods for boilerplate reduction.

```java
// Creates an item that automatically cancels click/drag events and hides tooltips
MenuStack border = MenuStack.of(ItemType.BLACK_STAINED_GLASS_PANE, true, true);

// Creates a stack with dedicated builder logic
MenuStack actionButton = MenuStack.builder()
        .content(ItemType.EMERALD)
        .click(e -> e.cancel())
        .build();
```

---

## Declarative String Layouts

The `MenuRecipe` engine parses multi-line strings into slot definitions, decoupling visual structure from Java logic. Ensure your layout string length exactly matches the capacity of your configured `MenuType`.

```java
import sh.miles.menukit.strings.MenuRecipe;
import sh.miles.menukit.strings.MenuStack;
import sh.miles.menukit.menu.SlotMenu;
import sh.miles.menukit.menu.MenuType;
import org.bukkit.entity.Player;
import org.bukkit.inventory.InventoryView;
import org.bukkit.inventory.ItemType;
import io.papermc.paper.datacomponent.DataComponentTypes;
import net.kyori.adventure.text.Component;

public class ClassSelectorMenu extends SlotMenu<InventoryView> {

    private static final MenuRecipe TEMPLATE = MenuRecipe.builder()
            .page(0, """
             BBBBBBBBB
             B---W---B
             BBBBBBBBB""")
            .map('B', MenuStack.of(ItemType.GRAY_STAINED_GLASS_PANE, true, true))
            .map('W', MenuStack.builder()
                    .content(ItemType.IRON_SWORD, item -> {
                        item.setData(DataComponentTypes.ITEM_NAME, Component.text("Warrior"));
                    })
                    .click(e -> e.cancel())
                    .build())
            .build();

    public ClassSelectorMenu(Player player) {
        super(player, p -> MenuType.GENERIC_9X3.create(p, Component.text("Classes")), 1);
    }

    @Override
    protected void reload(InventoryView view) {
        TEMPLATE.apply(this.getInventory());
    }
}
```

---

## State Management & Pagination

`PagedInventory` allows a single frontend `InventoryView` to represent multiple virtual pages. Slots are seamlessly swapped out when the active page is mutated.

```java
import sh.miles.menukit.menu.SlotMenu;
import sh.miles.menukit.menu.MenuType;
import org.bukkit.entity.Player;
import org.bukkit.inventory.InventoryView;
import org.bukkit.inventory.ItemType;
import io.papermc.paper.datacomponent.DataComponentTypes;
import net.kyori.adventure.text.Component;

public class HistoryMenu extends SlotMenu<InventoryView> {

    public HistoryMenu(Player player) {
        super(player, p -> MenuType.GENERIC_9X3.create(p, Component.text("Audit Log")), 2);
    }

    @Override
    protected void reload(InventoryView view) {
        
        // Define an item explicitly on Page 0
        this.getInventory().setItem(createSlot(builder -> builder
                .page(0)
                .index(13)
                .content(ItemType.PAPER, item -> item.setData(DataComponentTypes.ITEM_NAME, Component.text("Log Entry 1")))
                .disableInteractions()
        ));

        // Define an item explicitly on Page 1 at the exact same index
        this.getInventory().setItem(createSlot(builder -> builder
                .page(1)
                .index(13)
                .content(ItemType.PAPER, item -> item.setData(DataComponentTypes.ITEM_NAME, Component.text("Log Entry 2")))
                .disableInteractions()
        ));

        // Create a navigation control that spans all pages via the current page mutator
        this.getInventory().setItem(createSlot(builder -> builder
                .page(0)
                .index(26)
                .content(ItemType.ARROW, item -> item.setData(DataComponentTypes.ITEM_NAME, Component.text("Next Page")))
                .click(e -> {
                    e.cancel();
                    int next = this.getInventory().getCurrentPage(0) + 1;
                    if (next < this.getInventory().getPages()) {
                        this.getInventory().setCurrentPage(next);
                    }
                })
        ));
    }
}
```

### Targeted Page Updates
You can isolate pagination to specific slots, allowing you to maintain static controls (like borders or navigation arrows) while only paginating the interior grid.
```java
// Sets slots 10 through 16 to display content from virtual page 2
this.getInventory().setCurrentPageFor(2, new int[]{10, 11, 12, 13, 14, 15, 16});
```

---

## Reactive Component Updates

Because callbacks pass the `SlotMenu` instance context via `MenuEventCallback`, you can safely trigger UI updates reactively within the click event execution cycle without requiring total view reloads.

```java
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.ItemType;
import io.papermc.paper.datacomponent.DataComponentTypes;
import net.kyori.adventure.text.Component;
import sh.miles.menukit.slot.MenuSlot;

MenuSlot toggleButton = createSlot(builder -> builder
        .index(4)
        .content(ItemType.RED_DYE, i -> i.setData(DataComponentTypes.ITEM_NAME, Component.text("Disabled")))
        .click(e -> {
            e.cancel();
            
            // Rebuild or mutate the ItemStack
            ItemStack newState = ItemType.LIME_DYE.createItemStack();
            newState.setData(DataComponentTypes.ITEM_NAME, Component.text("Enabled"));
            
            // Fetch the slot and push the new content reactively
            MenuSlot slot = this.getInventory().getSlot(4);
            slot.setContent(newState);
        })
);
```

## Specialized Containers

Because `SlotMenu` is parameterized with `<V extends InventoryView>`, MenuKit is not restricted to standard chest grids. You can seamlessly construct functional menus for specialized containers like Anvils, Hoppers, or Droppers using the appropriate `MenuType`.

```java
import sh.miles.menukit.menu.SlotMenu;
import sh.miles.menukit.menu.MenuType;
import org.bukkit.entity.Player;
import org.bukkit.inventory.InventoryView;
import net.kyori.adventure.text.Component;

public class RepairMenu extends SlotMenu<AnvilView> {

    public RepairMenu(Player player) {
        // Initializes a 3-slot Anvil interface
        super(player, p -> MenuType.ANVIL.create(p, Component.text("Custom Repair")), 1);
    }

    @Override
    protected void reload(InventoryView view) {
        // Bind logic to the Anvil's specific slot indices (0: Left, 1: Right, 2: Result)
        this.getInventory().setItem(createSlot(builder -> builder
                .index(2)
                .click(e -> {
                    // Custom repair logic here
                    e.getPlayer().sendMessage(Component.text("Item Forged!"));
                })
        ));
    }
}
```
