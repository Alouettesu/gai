---
layout: post
title:  "A Date Picker Component in QML"
date:   2026-07-02 06:00:00 +0800
categories: date-picker qml
---

There are different approaches to building date picker components in a user interface. For example, Qt6 provides the [MonthGrid](https://doc.qt.io/qt-6/qml-qtquick-controls-monthgrid.html) component, which looks like a calendar: to pick a date, the user selects a year and month and then taps a day in the grid. A more modern approach is based on three spinning wheels — one each for the year, month, and day. This reduces the need for precise clicks: the user can swipe, drag, or scroll the mouse wheel without worrying about exact pointer placement, making date selection faster and more comfortable. The component works equally well in desktop, web, and mobile applications.

A ready-made solution is available in the [Felgo](https://felgo.com/doc/felgo-datepicker/) component library — if you are comfortable pulling in a large third-party dependency with many tightly coupled components. But if you need a minimal yet fully functional implementation that can be dropped straight into your project, this article is for you.

If you prefer to jump straight to the code, head over to the [repository](https://github.com/Alouettesu/qml-date-picker). Otherwise, read on — I will explain how the component works and how to build on top of it.

## SpinningWheel — the base spinning wheel component

The foundation of the picker is the `SpinningWheel` component. It implements a single spinning wheel that displays arbitrary data. The user can select an item by clicking, scrolling, or dragging. The component accepts any type of data model: numeric, `ListModel`, JavaScript array, or `QAbstractItemModel`, and supports custom background, highlight, and delegate components. For numeric models and JavaScript arrays the default delegate is sufficient. For more complex models such as `ListModel` or `QAbstractItemModel` — which expose multiple roles — a custom delegate is required. Let us look at examples using each model type.

All components are part of the Gai QML module, which lives in the imports/Gai folder of the repository.

### SpinningWheel properties

- `model: var` — Data model (numeric, JavaScript array, `ListModel`, or `QAbstractItemModel`).
- `currentIndex: int` — Currently selected index, zero-based.
- `delegate: Component` — Delegate for rendering items. The default delegate renders a `Text` element.
- `highlight: Component` — Component used to highlight the current item.
- `background: Component` — Background component.
- `font: font` — Font used by the default delegate.

### SpinningWheel signals

- `activated(int index)` — Emitted when the user changes the current selection. Not emitted when the selection is changed programmatically.
- `currentIndexChanged` — Emitted whenever the current selection changes, whether by the user or programmatically.

### A simple wheel with a numeric model

With a numeric model the default delegate renders numbers from `0` to `N-1`, where `N` is the value of the `model` property.

```qml
SpinningWheel {
    id: wheel1
    font: Qt.font({ family: "Arial", pointSize: 14 })
    model: 10
}
```

The delegate, background, and highlight are all optional. If omitted, the defaults are used.

### A wheel with a JavaScript array model

With an array model every element of the array is placed on the wheel.

```qml
SpinningWheel {
    id: wheel2
    font: Qt.font({ family: "Arial", pointSize: 14 })
    model: ["Apples", "Pears", "Bananas", "Peaches", "Grapes", "Watermelon"]
}
```

### A wheel with a ListModel

When using a role-based model (`ListModel` or `QAbstractItemModel`) the default delegate cannot render the items correctly, so a custom delegate is required. In the example below the model has two roles; they are accessible in the delegate as `model.role_name`.

```qml
SpinningWheel {
    id: wheel3
    font: Qt.font({ family: "Arial", pointSize: 14 })
    model: ListModel {
        ListElement { emoji: "🍎"; label: "Apple"  }
        ListElement { emoji: "🍐"; label: "Pear"   }
        ListElement { emoji: "🍌"; label: "Banana" }
        ListElement { emoji: "🍑"; label: "Peach"  }
        ListElement { emoji: "🍇"; label: "Grapes" }
        ListElement { emoji: "🍓"; label: "Berry"  }
    }
    delegate: Item {
        width: parent ? parent.width : 0
        height: parent ? parent.height : 0
        property color textColor: "#222"
        Row {
            anchors.centerIn: parent
            spacing: 6
            Text { text: model.emoji; font.pointSize: 14; verticalAlignment: Text.AlignVCenter }
            Text { text: model.label; font: root.font; color: parent.parent.textColor; verticalAlignment: Text.AlignVCenter }
        }
    }
}
```

Here is what all three wheels look like in action:

![SpinningWheel animation]({{ site.baseurl }}/images/SpinningWheel.gif)

## Date component wheels: DayPicker, MonthPicker, YearPicker

Sometimes you only need to pick a year, a month, or a day. For these cases the library provides dedicated single-wheel components: `YearPicker`, `MonthPicker`, and `DayPicker`.

- `currentYear`, `currentMonth`, `currentDay: int` — The currently selected year, month, or day.
- `range: object` — A JavaScript object with `from` and `to` attributes that defines the allowed range. When the range is changed (by assignment or binding) it is validated (`from ≤ to`) and may be rejected. If rejected, a message is printed to the console. If the current selection falls outside the new range it is clamped to the nearest boundary.
- `delegate: Component` — Custom delegate.
- `background: Component` — Custom background.
- `highlight: Component` — Custom highlight.
- `locale: Locale` — Locale used for month names.

```qml
YearPicker {
    currentYear: 2026
    range: { from: 2020, to: 2030 }
}

MonthPicker {
    currentMonth: 5  // June (0-indexed)
    range: { from: 0, to: 11 }
}

DayPicker {
    currentDay: 15
    range: { from: 1, to: 31 }
}
```

Here is the month picker in action:

![MonthPicker animation]({{ site.baseurl }}/images/MonthPicker.gif)

Although the individual date wheels are fully self-contained, you will most likely want to use all three together. That is exactly what the date picker component is for.

## The date picker

By grouping all three wheels together we get a full date picker component. You can set a date range, and the available range for each wheel is computed dynamically based on the current selection.

> For example, suppose the end date is 2028-06-06 and the current selection is 2027-05-05 — all twelve months are available in the month wheel. But if the current selection is 2028-05-05 with the same range, the month wheel only shows January through July.

The number of days in the day wheel is also adjusted automatically based on the selected month and the active date range.

### DatePicker properties

- `selectedDate: date` — The currently selected date.
- `dateRange: object` — The allowed date range (a JavaScript object with `begin` and `end` attributes).
- `locale: Locale` — Locale used for month names.
- `font: font` — Font used by the default delegates. Also used to calculate the width of each wheel so that all years, months, and days fit with some padding.
- `delegate: Component` — A shared delegate applied to all three wheels.
- `background: Component` — A shared background applied to each wheel individually.
- `highlight: Component` — Highlight component applied to all three wheels.
- `highlightYear: Component` — Highlight component for the year wheel only.
- `highlightMonth: Component` — Highlight component for the month wheel only.
- `highlightDay: Component` — Highlight component for the day wheel only.

The `highlight` property sets a single highlight component for all three wheels at once. The individual properties `highlightYear`, `highlightMonth`, and `highlightDay` let you override the highlight for each wheel separately. Assigning `highlight` overwrites all three; assigning an individual property overwrites only the corresponding wheel. This is useful when you want rounded corners on the left edge of the year wheel and the right edge of the day wheel, as in the Neon2 theme.

### DatePicker signals

- `activated(date selected)` — Emitted when the user changes the selected date. Not emitted when the date is changed programmatically.
- `selectedDateChanged` — Emitted whenever the selected date changes, whether by the user or programmatically.

### Basic usage

```qml
DatePicker {
    id: startDatePicker
    selectedDate: new Date(2026, 0, 1)
    dateRange: {
        begin: new Date(2020, 0, 1),
        end: new Date(2030, 11, 31)
    }
}
```

![DatePicker animation]({{ site.baseurl }}/images/DatePicker.gif)

### Themes

Themes are not part of the `Gai` module itself, but they are straightforward to create. The demo application in `main.qml` includes several examples that show the full range of styling options. Below is the Neon theme for `DatePicker`. For small projects you can declare the components inline as shown here; in larger codebases it is cleaner to define them as separate `Component` objects. The image below shows all six themes side by side.

```qml
import QtQuick
import QtQuick.Window
import Gai

Window {
    id: root
    width: 400
    height: 500
    visible: true
    title: "Neon DatePicker"
    color: "#0a0a1a"

    // This font is used both for rendering text and for calculating
    // the width of the year, month, and day wheels.
    property font appFont: Qt.font({ family: "Arial", pointSize: 14 })

    DatePicker {
        anchors.centerIn: parent
        width: parent.width * 0.8
        height: parent.height * 0.8
        font: root.appFont

        // Initial selection: 15 June 2026
        selectedDate: new Date(2026, 5, 15)
        // Allowed range: 1 January 2020 – 31 December 2030
        dateRange: ({ begin: new Date(2020, 0, 1), end: new Date(2030, 11, 31) })

        // Delegate that renders each item as text
        delegate: Text {
            text: modelData
            font: root.appFont
            color: "#00ffcc"
            horizontalAlignment: Text.AlignHCenter
            verticalAlignment: Text.AlignVCenter
        }

        // Background with gradient fade at the top and bottom edges
        background: Rectangle {
            color: "#0a0a1a"
            Rectangle {
                anchors { top: parent.top; left: parent.left; right: parent.right }
                height: parent.height * 0.35
                gradient: Gradient {
                    GradientStop { position: 0.0; color: "#e00a0a1a" }
                    GradientStop { position: 1.0; color: "#000a0a1a" }
                }
                z: 1
            }
            Rectangle {
                anchors { bottom: parent.bottom; left: parent.left; right: parent.right }
                height: parent.height * 0.35
                gradient: Gradient {
                    GradientStop { position: 0.0; color: "#000a0a1a" }
                    GradientStop { position: 1.0; color: "#e00a0a1a" }
                }
                z: 1
            }
        }

        // Highlight indicator for the currently selected item
        highlight: Rectangle { color: "#1100ffcc"; border.color: "#00ffcc"; border.width: 1; radius: 6 }
    }
}
```

![All themes]({{ site.baseurl }}/images/ThemesCompare.png)

## Integrating DatePicker into your project

The components were developed and tested with:

- Qt 6.x (Core, Gui, Quick, Qml modules);
- CMake 3.16 or higher;
- A C++17-compatible compiler.

They may also work with earlier or later versions.

The repository is structured as follows:

```
DatePickerDemo/
├── qml/
│   └── main.qml              # Demo application UI
├── src/
│   └── main.cpp              # Application entry point
├── imports/
│   └── Gai/
│       ├── SpinningWheel.qml # Base spinning wheel component
│       ├── DatePicker.qml    # Main date picker component
│       ├── YearPicker.qml    # Year selection component
│       ├── MonthPicker.qml   # Month selection component
│       ├── DayPicker.qml     # Day selection component
│       ├── qmldir            # QML module definition
│       └── CMakeLists.txt    # Build configuration
├── CMakeLists.txt            # Project build configuration
├── DatePickerDemo.qmlproject # Qt Design Studio project file
├── qtquickcontrols2.conf     # Qt Quick Controls configuration
└── resources.qrc             # Resource file
```

The `imports` folder contains the `Gai` QML module, which provides all the components. The module includes a `qmldir` file for Qt Design Studio and a `CMakeLists.txt` for building. Copy the `Gai` folder into your project and link the module:

```cmake
add_subdirectory(imports/Gai)
target_link_libraries(<your-target-name> PRIVATE
    Gaiplugin
)
```

> A note on `qmldir`: the primary way to use the module is to build it as a static library with all QML files embedded in its resources. In that case CMake generates its own `qmldir` inside the resources, so the `qmldir` file in the repository is not used at runtime. However, Qt Design Studio needs to read the module's exported components from a local `qmldir` file when you open the project in the IDE. There are of course other deployment options — QML modules can live on the local filesystem or even on a network — but they are not covered by this repository.

Import the `Gai` module wherever you want to use the components:

```qml
import Gai
Item {
    DatePicker {
        selectedDate: new Date(2026, 0, 1)
        dateRange: {
            begin: new Date(2020, 0, 1),
            end: new Date(2030, 11, 31)
        }
    }
}
```

That is all it takes to add the module to your project and enable live preview in Qt Design Studio.

## Conclusion

We have covered why and how to use a QML date picker, how it is built internally, and how to integrate it into your own project. We looked at customizing its appearance and behavior, and walked through the integration steps. The component is a good fit for user profile forms, booking flows, event planning applications, and any other context where a date needs to be selected.

Future development could include time selection support and additional customization options. If you have ideas for improving the component, feel free to open an issue or submit a pull request. Thanks for reading!